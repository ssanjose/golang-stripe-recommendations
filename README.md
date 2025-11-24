# golang-stripe-recommendations
Inspired(literally copied from) by https://github.com/t3dotgg/stripe-recommendations

Check out the above link for a full break down, but it is essentially the same.

Feel free to suggest edits to this repo, I'll make sure to take a look.

### Pre-requirements

- Golang
- Some type of Golang backend(I used chi for this tutorial)
- Working auth (that is verified on your Golang backend usually through a middleware)
- A KV store (I use Redis), but any KV will work)

### General philosophy

IMO, the biggest issue with Stripe is the "split brain" it inherently introduces to your code base. When a customer checks out, the "state of the purchase" is in Stripe. You're then expected to track the purchase in your own database via webhooks.

There are [over 258 event types](https://docs.stripe.com/api/events/types). They all have different amounts of data. The order you get them is not guaranteed. None of them should be trusted. It's far too easy to have a payment be failed in stripe and "subscribed" in your app.

These partial updates and race conditions are obnoxious. I recommend avoiding them entirely. My solution is simple: _a single `syncStripeSubscription(customerId: string)` function that syncs all of the data for a given Stripe customer to your KV_.

The following is how I (mostly) avoid getting Stripe into these awful split states.

## The Flow

This is a quick overview of the "flow" I recommend. More detail below. Even if you don't copy my specific implementation, you should read this. _I promise all of these steps are necessary. Skipping any of them will make life unnecessarily hard_

1. **FRONTEND:** "Subscribe" button should call a `"generate-stripe-checkout"` endpoint onClick
1. **USER:** Clicks "subscribe" button on your app
1. **BACKEND:** Create a Stripe customer
1. **BACKEND:** Store binding between Stripe's `customerId` and your app's `userId`
1. **BACKEND:** Create a "checkout session" for the user
   - With the return URL set to a dedicated `/success` route in your app
1. **USER:** Makes payment, subscribes, redirects back to `/success`
1. **FRONTEND:** On load, triggers a `handleSubscriptionSuccessHandler` function on backend (hit an API for redundancy)
1. **BACKEND:** Uses `userId` to get Stripe `customerId` from KV
1. **BACKEND:** Calls `syncStripeSubscription` with `customerId`
1. **FRONTEND:** After sync succeeds, redirects user to wherever you want them to be :)
1. **BACKEND:** On [_all relevant events_](#allowedevent), calls `syncStripeSubscription` with `customerId`

This might seem like a lot. That's because it is. But it's also the simplest Stripe setup I've ever seen work.

Let's go into the details on the important parts here.

### Checkout flow

The key is to make sure **you always have the customer defined BEFORE YOU START CHECKOUT**. The ephemerality of "customer" is a straight up design flaw and I have no idea why they built Stripe like this.

```golang
type CreateCheckoutSessionPayload struct {
	Tier string `json:"tier" validate:"required,oneof=free standard premium"`
}

type CheckoutSessionResponse struct {
	URL string `json:"url"`
}

func (app *application) createCheckoutSessionHandler(w http.ResponseWriter, r *http.Request) {
	var payload CreateCheckoutSessionPayload
	if err := readJSON(w, r, &payload); err != nil {
		app.badRequestResponse(w, r, err)
		return
	}

	if err := Validate.Struct(payload); err != nil {
		app.badRequestResponse(w, r, err)
		return
	}

	user, _ := getUserFromCtx(r)
	if user == nil {
		app.unauthorizedErrorResponse(w, r, nil)
		return
	}

	if user.Email == "" {
		app.badRequestResponse(w, r, store.ErrStripeEmailRequired)
		return
	}

	tier := payload.Tier
	if tier == "" {
		app.badRequestResponse(w, r, store.ErrSubscriptionTierNotFound)
		return
	}

	// get tier price ID from the subscription tiers
	priceID, err := store.GetSubscriptionTierPriceID(tier)
	if err != nil {
		if err == store.ErrSubscriptionTierNotFound {
			app.badRequestResponse(w, r, err)
			return
		}
		app.internalServerError(w, r, err)
		return
	}

	// Retrieve Stripe customer from cache
	stripeSubscription, err := app.cacheStorage.Subscriptions.Get(r.Context(), user.ID)
	if err != nil {
		app.internalServerError(w, r, err)
		return
	}

	// If no Stripe customer found, create a new one
	if stripeSubscription == nil || stripeSubscription.CustomerID == "" {
		// Create a new Stripe customer
		customerParams := &stripe.CustomerParams{
			Email: stripe.String(user.Email),
			Metadata: map[string]string{
				"user_id": fmt.Sprint(user.ID),
			},
		}
		c, err := customer.New(customerParams)
		if err != nil {
			app.internalServerError(w, r, err)
			return
		}

		// Assign the newly created customer ID to the subscription
		stripeSubscription = &store.Subscription{
			CustomerID: c.ID,
			UserID: user.ID,
		}

		// Cache the Stripe customer information(only the created subscription ID and the userID, so we can retrieve it later)
		err = app.cacheStorage.Subscriptions.Set(r.Context(), stripeSubscription)
		if err != nil {
			app.internalServerError(w, r, err)
			return
		}
	}

	params := &stripe.CheckoutSessionParams{
		LineItems: []*stripe.CheckoutSessionLineItemParams{
			{
				Price:    stripe.String(priceID),
				Quantity: stripe.Int64(1),
			},
		},
		// Use the existing customer or the newly created one for the session
		Customer:   stripe.String(stripeSubscription.CustomerID),
		Mode:       stripe.String(string(stripe.CheckoutSessionModeSubscription)),
		SuccessURL: stripe.String(app.config.stripe.stripeSuccessURL),
		CancelURL:  stripe.String(app.config.stripe.stripeCancelURL),
		// Add metadata to the subscription that will be created
		SubscriptionData: &stripe.CheckoutSessionSubscriptionDataParams{
			Metadata: map[string]string{
				"user_id": fmt.Sprint(user.ID),
			},
		},
	}

	s, err := session.New(params)
	if err != nil {
		app.internalServerError(w, r, err)
		return
	}

	response := CheckoutSessionResponse{
		URL: s.URL,
	}

	if err := app.jsonResponse(w, http.StatusOK, response); err != nil {
		app.internalServerError(w, r, err)
		return
	}
}
```

### syncStripeSubscription

This is the function that syncs all of the data for a given Stripe customer to your KV. It will be used in both your `/success` endpoint and in your `/api/stripe` webhook handler.

The Stripe api returns a ton of data, much of which can not be serialized to JSON. I've selected the "most likely to be needed" chunk here for you to use, and there's a [type definition later in the file](#custom-stripe-subscription-type).

Your implementation will vary based on if you're doing subscriptions or one-time purchases.

```golang
func (app *application) syncStripeSubscription(ctx context.Context, customerID string, sync bool) error {
	subscriptions := subscription.List(&stripe.SubscriptionListParams{
		Customer: &customerID,
		ListParams: stripe.ListParams{
			Limit: stripe.Int64(1),
		},
		Status: stripe.String("all"),
		Expand: stripe.StringSlice([]string{"data.default_payment_method"}),
	})

	subscriptionList := subscriptions.SubscriptionList()
	if len(subscriptionList.Data) == 0 {
		return fmt.Errorf("no subscriptions found for customer ID: %v", customerID)
	}

	subscription := subscriptionList.Data[0]

	userIDStr, exists := subscription.Metadata["user_id"]
	if !exists || userIDStr == "" {
		return fmt.Errorf("user_id not found in subscription metadata for customer: %v", customerID)
	}

	userIDInt64, err := strconv.ParseInt(userIDStr, 10, 64)
	if err != nil {
		return fmt.Errorf("invalid user ID: %v", userIDStr)
	}

	if userIDInt64 <= 0 {
		return fmt.Errorf("invalid user ID: %v", userIDStr)
	}

	var brand, last4 *string
	if subscription.DefaultPaymentMethod != nil && subscription.DefaultPaymentMethod.Card != nil {
		brandValue := string(subscription.DefaultPaymentMethod.Card.Brand)
		brand = &brandValue
		last4 = &subscription.DefaultPaymentMethod.Card.Last4
	}

	var currentPeriodEnd, currentPeriodStart *int64
	if subscription.LatestInvoice != nil {
		currentPeriodEnd = &subscription.LatestInvoice.PeriodEnd
		currentPeriodStart = &subscription.LatestInvoice.PeriodStart
	}

	var priceID string
	if len(subscription.Items.Data) > 0 && subscription.Items.Data[0].Price != nil {
		priceID = subscription.Items.Data[0].Price.ID
	} else {
		return fmt.Errorf("no price ID found for subscription: %v", subscription.ID)
	}

	customerSubscription := &store.Subscription{
		CustomerID:             subscription.Customer.ID,
		SubscriptionID:         subscription.ID,
		UserID:                 userIDInt64,
		Status:                 string(subscription.Status),
		PriceID:                priceID,
		CurrentPeriodEnd:       currentPeriodEnd,
		CurrentPeriodStart:     currentPeriodStart,
		CancelAtPeriodEnd:      subscription.CancelAtPeriodEnd,
		PaymentMethodCardBrand: brand,
		PaymentMethodCardLast4: last4,
	}

	if sync {
		return app.cacheStorage.Subscriptions.Set(ctx, customerSubscription)
	}

	app.cacheStorage.Subscriptions.AsyncSet(customerSubscription)

	return nil
}
```

### `/success` endpoint

> [!NOTE]
> While this isn't 'necessary', there's a good chance your user will make it back to your site before the webhooks do. It's a nasty race condition to handle. Eagerly calling syncStripeSubscription will prevent any weird states you might otherwise end up in

This is the page that the user is redirected to after they complete their checkout. For the sake of simplicity, I'm going to implement it as a `get` route that redirects them. In my apps, I do this with a server component and Suspense, but I'm not going to spend the time explaining all that here.

```golang
func (app *application) handleSubscriptionSuccessHandler(w http.ResponseWriter, r *http.Request) {
	user, _ := getUserFromCtx(r)
	if user == nil {
		app.unauthorizedErrorResponse(w, r, nil)
		return
	}

	queryParams := r.URL.Query()
	billingStatus := queryParams.Get("billing")
	if billingStatus != "success" {
		app.badRequestResponse(w, r, fmt.Errorf("invalid billing status"))
		return
	}

	// Sync the latest subscription data from Stripe
	subscription, err := app.cacheStorage.Subscriptions.Get(r.Context(), user.ID)
	if err != nil {
		app.internalServerError(w, r, err)
		return
	}

	if subscription == nil || subscription.CustomerID == "" {
		app.badRequestResponse(w, r, fmt.Errorf("no subscription found for user"))
		return
	}

	err = app.syncStripeSubscription(r.Context(), subscription.CustomerID, true)
	if err != nil {
		app.logger.Errorw("error syncing stripe subscription after checkout", "error", err, "user_id", user.ID)
	}

	redirectURL := fmt.Sprintf("%s/user/dashboard?billing=success", app.config.frontendURL)

	app.redirectResponse(w, r, nil, redirectURL, http.StatusSeeOther)
}
```

Notice how I'm not using any of the `CHECKOUT_SESSION_ID` stuff? That's because it sucks and it encourages you to implement 12 different ways to get the Stripe state. Ignore the siren calls. Have a SINGLE `syncStripeSubscription` function. It will make your life easier.

### `/api/stripe` (The Webhook)

This is the part everyone hates the most. I'm just gonna dump the code and justify myself later.

```golang
func (app *application) stripeWebhookHandler(w http.ResponseWriter, r *http.Request) {
	// Handle Stripe webhook events here
	// You can use the Stripe Go library to parse and handle the events
	// get stripe signature from headers
	signature := r.Header.Get("Stripe-Signature")
	if signature == "" {
		app.badRequestResponse(w, r, fmt.Errorf("[STRIPE SIGNATURE] is missing"))
		return
	}

	// check if signature is of type string
  // builtin to golang, you don't have to type this
	if _, ok := interface{}(signature).(string); !ok {
		app.badRequestResponse(w, r, fmt.Errorf("[STRIPE SIGNATURE] is not of type string"))
		return
	}

	// decode the request body
	maxBytes := int64(65536)
	r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
	payload, err := io.ReadAll(r.Body)
	if err != nil {
		app.badRequestResponse(w, r, fmt.Errorf("webhook error while parsing request body: %v", err))
		return
	}

	event, err := webhook.ConstructEvent(
		payload,
		signature,
		app.config.stripe.webhookSecret,
	)
	if err != nil {
		app.badRequestResponse(w, r, fmt.Errorf("webhook signature verification failed: %v", err))
		return
	}

	if !allowedEvent(event) {
		app.badRequestResponse(w, r, fmt.Errorf("error processing event: %s", event.Type))
		return
	}

	customerIDInterface, exists := event.Data.Object["customer"]
	if !exists {
		app.internalServerError(w, r, fmt.Errorf("customer ID is missing in the event data"))
		return
	}

	customerID, ok := customerIDInterface.(string)
	if !ok || customerID == "" {
		app.internalServerError(w, r, fmt.Errorf("customer ID is invalid or missing in the event data"))
		return
	}

	err = app.syncStripeSubscription(r.Context(), customerID, false)
	if err != nil {
		app.internalServerError(w, r, err)
		return
	}

	writeJSON(w, http.StatusOK, map[string]bool{"received": true})
}
```

### `allowedEvent`

This is the function called in the endpoint that actually takes the Stripe event and updates the KV.

```golang
func allowedEvent(eventType stripe.Event) bool {
	allowedEvents := []string{
		"checkout.session.completed",
		"customer.subscription.created",
		"customer.subscription.updated",
		"customer.subscription.deleted",
		"customer.subscription.paused",
		"customer.subscription.resumed",
		"customer.subscription.pending_update_applied",
		"customer.subscription.pending_update_expired",
		"customer.subscription.trial_will_end",
		"invoice.paid",
		"invoice.payment_succeeded",
		"invoice.payment_failed",
		"invoice.payment_action_required",
		"invoice.upcoming",
		"invoice.marked_uncollectible",
		"payment_intent.succeeded",
		"payment_intent.payment_failed",
		"payment_intent.canceled",
	}

	for _, event := range allowedEvents {
		if event == string(eventType.Type) {
			return true
		}
	}
	return false
}

```

### Custom Stripe subscription type

```golang
package store

type Subscription struct {
	CustomerID             string  `json:"customer_id"`               // customer ID in Stripe
	SubscriptionID         string  `json:"subscription_id"`           // subscription ID in Stripe
	Status                 string  `json:"status"`                    // subscription status
	PriceID                string  `json:"price_id"`                  // price ID
	UserID                 int64   `json:"user_id"`                   // user ID
	CurrentPeriodEnd       *int64  `json:"current_period_end"`        // end of the current billing period
	CurrentPeriodStart     *int64  `json:"current_period_start"`      // start of the current billing period
	CancelAtPeriodEnd      bool    `json:"cancel_at_period_end"`      // whether the subscription will cancel at period end
	PaymentMethodCardBrand *string `json:"payment_method_card_brand"` // card brand of the payment method
	PaymentMethodCardLast4 *string `json:"payment_method_card_last4"` // last 4 digits of the payment method
}
```

## More Pro Tips

Gonna slowly drop more things here as I remember them.

### DISABLE "CASH APP PAY".

I'm convinced this is literally just used by scammers. over 90% of my cancelled transactions are Cash App Pay.
![image](https://github.com/user-attachments/assets/c7271fa6-493c-4b1c-96cd-18904c2376ee)

### ENABLE "Limit customers to one subscription"

This is a really useful hidden setting that has saved me a lot of headaches and race conditions. Fun fact: this is the ONLY way to prevent someone from being able to check out twice if they open up two checkout sessions 🙃 More info [in Stripe's docs here](https://docs.stripe.com/payments/checkout/limit-subscriptions)

## Things that are still your problem

While I have solved a lot of stuff here, in particular the "subscription" flows, there are a few things that are still your problem. Those include...

- Managing `STRIPE_SECRET_KEY` and `STRIPE_PUBLISHABLE_KEY` env vars for both testing and production
- Managing `STRIPE_PRICE_ID`s for all subscription tiers for dev and prod (I can't believe this is still a thing)
- Exposing sub data from your KV to your user (a dumb endpoint is probably fine)
- Tracking "usage" (i.e. a user gets 100 messages per month)
- Managing "free trials"
  ...the list goes on

Regardless, I hope you found some value in this doc.
