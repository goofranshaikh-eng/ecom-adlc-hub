---
key: PER-161
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: A shopper browses a catalog, checks out, and receives a paid order
---

## Job

When a client wants something the store sells, it finds the product, pays the price it was shown, and can look the order up later. All of this is HTTP APIs. There is no storefront and no UI.

## Who

* Client: any caller of the APIs. It may be a guest or signed in.
* Store operator: can see that a paid order exists through the order API. No admin UI in this job.

## What exists when this is done

A client can search and fetch a product, add it to a cart, check out, pay, and fetch the resulting order. Each capability is its own service. No service owns another service's data.

| Service | Owns |
| -- | -- |
| identity | Sign-in and the shopper id other services store. Guests need no account |
| catalog | Products, price, and whether an item can be sold |
| cart | The client's current items and the total it sees |
| checkout | Turning a cart into an order at the price shown |
| payment | Charging that amount and reporting paid or refused |
| order | The paid order the client can look up |

## Acceptance criteria

 1. **Find a product**

    **Given** the catalog has a sellable product

    **When** the client searches for it and fetches it

    **Then** the response includes its name, price, and that it can be added to the cart
 2. **Unsellable product cannot be added**

    **Given** a product that cannot be sold

    **When** the client fetches it

    **Then** the price is still returned and add-to-cart is refused with a reason
 3. **Cart total is the catalog price**

    **Given** a sellable product

    **When** the client adds a quantity

    **Then** the cart line shows that quantity and the total equals catalog price times quantity
 4. **Quantity change recomputes the total**

    **Given** a cart with a line

    **When** the client changes the quantity, including to zero

    **Then** the line updates or disappears and the total matches the remaining lines
 5. **Guest checkout**

    **Given** a cart and no account

    **When** the client checks out with a contact email and a ship-to address

    **Then** checkout asks payment for the cart total just returned
 6. **Signed-in checkout uses the same total**

    **Given** a signed-in shopper with a cart

    **When** the client checks out

    **Then** the amount sent to payment equals the cart total, and the order is stored against that shopper id
 7. **Payment refused**

    **Given** a cart at checkout

    **When** payment refuses the charge

    **Then** no order is created, the cart is unchanged, and the response says the payment was refused
 8. **Payment accepted**

    **Given** a cart at checkout

    **When** payment accepts the charge

    **Then** an order exists with the lines, the total charged, a payment reference, and the ship-to address, and the cart is empty
 9. **Look up the order**

    **Given** a paid order

    **When** the client fetches it

    **Then** the response includes the order id, lines, total, and address that were stored
10. **Services do not share databases**

    **Given** a completed purchase
    **When** each service is inspected
    **Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data
11. **Swagger for every service**

    **Given** a running service
    **When** a person opens that service's Swagger UI
    **Then** every endpoint for that service is listed and can be called from the page

## Boundaries

* No storefront, pages, or other UI. Swagger UI is the only browser surface.
* Must not change: catalog price rules, tax, and shipping cost. v1 charges the catalog total with no tax and no shipping fee.
* Must not change: how an item becomes sellable. This job reads that flag. It does not receive stock or edit the catalog.
* One payment provider. The client sees paid or refused, not the provider's raw response.
* Money is the catalog currency, in minor units, and the same amount crosses cart, checkout, payment, and order.

## Notes

* Assumption: one merchant, one currency.
* Assumption: guest checkout needs an email and a ship-to address, and no account is created for them.
* Assumption: the payment provider is a test double that can accept or refuse a charge.
* Assumption: v1 is one product repo, hub key `app`. The service names above are separate backend processes inside that repo.
* Parked: a storefront, reviews, search ranking, recommendations, coupons, tax, shipping rates, returns, multi-vendor sellers, subscriptions, and an operator UI.
*
