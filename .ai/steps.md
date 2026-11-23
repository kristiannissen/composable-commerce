# Step vocabulary
Steps speak in fixtures, events, statuses, HTTP, stored values, external
responses. Never page text, CSS or internal object state.

Given:
- Given a merchant :cvr exists with status :status
- Given merchant :cvr has discount :percent percent
- Given product :sku exists with price :amount DKK
- Given product :sku has stock :qty, MOQ :moq, pack size :pack
- Given product :sku belongs to category :slug
- Given variant :sku of product :sku exists
- Given cart :line total line items :lines
- Given order :id has status :status
- Given a signed session for merchant :cvr
- Given the clock is :time
- Given :vendor is down
- Given :vendor will respond :status with :fixture

When:
- When provider :cvr is validated via VIES
- When merchant :cvr is approved
- When product :sku is updated with price :amount DKK
- When cart :id is submitted by merchant :cvr
- When the outbox dispatcher runs
- When listener :listener handles event :event
- When event :event is redelivered
- When I call :method api/:path with body :json (signed, merchant :cvr)
- When I call :method api/:path anonymously

Then:
- Then the last response status is :status
- Then the last response body has :path = :value
- Then event :event is published with :path = :value
- Then no event is published
- Then order :id has status :status
- Then the outbox for :event is empty
- Then :event is processed exactly :n times
- Then a request was made to :vendor containing :value
- Then no request was made to :vendor
- Then merchant :cvr sees price :amount DKK for product :sku

Missing step -> needed line under "# New steps" in the feature, waits
for the human. Expect this rarely after the first modules.
