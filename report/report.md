## TicketMaster Smart Contract Report

Authors: Chad Rossouw and Murphy Lee

### TicketMaster (Sol) Smart Contract Plan

##### Variables

| **Variable**          | **Type**                         | **Description**                                                                |
| --------------------- | -------------------------------- | ------------------------------------------------------------------------------ |
| `tcoin`               | `TCoin`                          | Reference to the deployed ERC20 token contract used for ticket payments        |
| `_tokenIdCounter`     | `uint256`                        | Tracks the incremental ID for newly minted tickets                             |
| `eventIdCounter`      | `uint256`                        | Tracks how many events have been created (acts as a unique event ID)           |
| `secondaryRoyaltyBps` | `uint96`                         | Secondary royalty percentage (basis points). Applied on secondary ticket sales |
| `platformFeeBps`      | `uint96`                         | Platform fee percentage (basis points). Applied on every ticket sale           |
| `_treasuryAddress`    | `address`                        | Address where platform fees are sent                                           |
| `events`              | `mapping(uint256 => EventInfo)`  | Stores an `EventInfo` struct for each eventId                                  |
| `tickets`             | `mapping(uint256 => TicketData)` | Stores a `TicketData` struct for each ticket ID                                |

##### Structs
| EventInfo        | Type        | Description                                    |
| ---------------- | ----------- | ---------------------------------------------- |
| `id`             | `uint256`   | The unique event identifier                    |
| `organizer`      | `address`   | The address of the event creator               |
| `name`           | `string`    | Name/title of the event                        |
| `description`    | `string`    | Description or details about the event         |
| `startTimestamp` | `uint256`   | Unix timestamp for event start time            |
| `endTimestamp`   | `uint256`   | Unix timestamp for event end time              |
| `location`       | `string`    | Location or venue of the event                 |
| `ticketIds`      | `uint256[]` | Array of ticket IDs associated with this event |

| TicketInfo | Type      | Description                      |
| ---------- | --------- | -------------------------------- |
| `price`    | `uint256` | Price (in TCOIN) for this ticket |
| `seat`     | `string`  | Seat label or identifier         |

| TicketData | Type      | Description                                             |
| ---------- | --------- | ------------------------------------------------------- |
| `id`       | `uint256` | The ERC721 token ID                                     |
| `price`    | `uint256` | The ticket price in TCOIN                               |
| `seat`     | `string`  | Seat label                                              |
| `eventId`  | `uint256` | Reference to the event this ticket belongs to           |
| `forSale`  | `bool`    | Indicates whether the ticket is currently on the market |

##### Events

| Event          | Parameters                                                                  | Description                                     |
| -------------- | --------------------------------------------------------------------------- | ----------------------------------------------- |
| `EventCreated` | `(uint id)`                                                                 | Emitted when a new event is created             |
| `TicketSold`   | `(_from address, _to address, id uint256, price uint256, isSecondary bool)` | Emitted when a ticket is bought                 |
| `PauseToggled` | `(bool paused)`                                                             | Emitted when the contract is paused or unpaused |

##### Functions

|**Function**|**Parameters**|**Visibility**|**Description**|
|---|---|---|---|
|**constructor(...)**|`_name, _symbol, _treasuryAddr, _tcoinAddr, _platformFeeBps`|`public`|Initializes the NFT contract name and symbol, sets treasury address, TCOIN address, fee config, etc.|
|**createEvent(...)**|`name, description, location, startTimestamp, endTimestamp, ticketDetails[], ticketURIs[]`|`external`|Organizer creates a new event with multiple tickets. Mints NFTs to the organizer, sets ticket prices, URIs.|
|**setEventName(...)**|`eventId, name`|`public`|Updates an event’s `name`; restricted to the event organizer|
|**setEventStart(...)**|`eventId, ts`|`public`|Updates an event’s `startTimestamp`|
|**setEventEnd(...)**|`eventId, ts`|`public`|Updates an event’s `endTimestamp`|
|**setEventLocation(...)**|`eventId, location`|`public`|Updates an event’s `location`|
|**setEventDescription(...)**|`eventId, description`|`public`|Updates an event’s `description`|
|**listTicketForSale(...)**|`ticketId, price`|`external`|Sets a ticket’s `forSale = true` and updates its price; only the ticket owner can call|
|**buyTicket(...)**|`ticketId`|`external`|Buyer purchases the ticket using TCOIN. Contract handles fees, royalties, and ownership transfer|
|**getTicketIds(...)**|`eventId`|`public view`|Returns the array of ticket IDs associated with a particular event|
|**pause()** / **unpause()**|_none_|`public`|Pauses/unpauses the `buyTicket` and other sensitive actions; `onlyOwner`|
|**setSecondaryRoyalty(...)**|`numerator (uint96)`|`public`|Updates `secondaryRoyaltyBps` if sum with platform fee <= 10000; `onlyOwner`|
|**setPlatformFee(...)**|`bps (uint96)`|`public`|Sets `platformFeeBps`; must keep combined fee <= 10000; `onlyOwner`|
|**changeTreasury(...)**|`newAddr`|`public`|Changes the treasury address used to collect platform fees; `onlyOwner`|
|**_feeDenominator()**|_none_|`private pure`|Returns `10000` (the basis points denominator)|
|**supportsInterface(...)**|`interfaceId`|`public view`|Overridden to combine ERC721, ERC721Enumerable, and any other inherited interfaces|
|**tokenURI(...)**|`tokenId`|`public view`|Overridden to retrieve URI from `ERC721URIStorage`|
|**_increaseBalance(...)**, **_update(...)**|various|`internal`|Low-level overrides from multiple inheritance (ERC721, ERC721Enumerable)|

### TCoin (Sol) Smart Contract Plan
|**Variable**|**Type**|**Description**|
|---|---|---|
|`RATE`|`uint256`|1 ETH = 10,000 TCOIN conversion rate|
|`MIN_PURCHASE`|`uint256`|Minimum ETH (0.001) to buy TCOIN|

|**Function**|**Description**|
|---|---|
|**constructor()**|Initializes TCOIN, mints an initial supply to deployer|
|**mint(...)**|Owner-only function to mint TCOIN to a given address|
|**buy()**|Exchanges ETH for TCOIN based on `RATE`, requires `msg.value >= MIN_PURCHASE`|
|**sell(amount)**|Burns TCOIN from caller in exchange for ETH (contract must have enough ETH)|
|**withdraw(amount)**|Owner-only function to withdraw ETH from the contract|

### Smart Contract Plan and Logic

Below, we outline how we structured the **TicketMaster** and **TCoin** contracts.

#### TicketMaster Contract

**Purpose & Design Rationale**  
We opted for **TicketMaster** to be an ERC721-based contract because each ticket is a unique NFT. This lets us leverage built-in functionality like `ownerOf` to track ownership, as well as standard NFT marketplaces (eg; OpenSea) and wallets that understand ERC721 tokens.

**Interfaces and Inheritance**
- **`ERC721Enumerable`** – Provides enumeration features (e.g., `totalSupply`, `tokenByIndex`, `tokenOfOwnerByIndex`). This makes it straightforward to list or iterate over all minted tickets, or see which tokens a specific user holds (eg; useful for `/my-tickets` page on the frontend).
- **`ERC721URIStorage`** – Allows us to associate a metadata URI with each token. We store additional data (images, seat info) off-chain via Pirana, referencing it by URI on-chain.
- **`Ownable`** – Enforces a single “owner” with exclusive rights to admin functions (like setting fees or pausing).
- **`Pausable`** – Lets us temporarily pause critical functions, such as `buyTicket`, in case of emergencies or upgrades.
- **`ReentrancyGuard`** – Prevents certain re-entrancy attacks, especially since `buyTicket` involves external token transfers.

**Key Variables**
- `tcoin (TCoin)`: Holds a reference to our ERC20-based TCOIN contract, which we use as the currency for ticket purchases.
- `_tokenIdCounter` / `eventIdCounter`: Two separate counters. One tracks the next ticket ID (because each minted ticket is a unique NFT), and the other tracks how many events have been created. This helps with assigning event/ticket IDs upon creation, for obvious reasons.
- `platformFeeBps` & `secondaryRoyaltyBps`: Store fee rates in **basis points** (1 = 0.01%). We limit them so that the combined total never exceeds 10,000 (i.e., 100%).
- `_treasuryAddress`: The address that collects platform fees.
- `events` & `tickets`: Mappings that store `EventInfo` structs (per event) and `TicketData` structs (per ticket). This allows quick lookups of event details or a ticket’s state (price, seat, etc.).

**Core Functions**
1. **`createEvent(...)`**
    - The organizer supplies arrays of `TicketInfo` (price & seat) and `ticketURIs` (IPFS metadata URIs).
    - We loop to mint each NFT (via `_safeMint`) and store the relevant ticket info in `tickets`
    - This design ensures each ticket is immediately owned by the event organizer, letting them manage listings or transfers.
2. **`buyTicket(ticketId)`**
    - The buyer must have enough TCOIN and must have approved this contract to spend it.
    - We handle the sale logic:
        - Calculate platform fee plus any secondary royalty if the seller is not the event organizer.
        - Distribute proceeds.
        - Transfer ownership of the ticket from seller to buyer.
    - Using an internal fee function (`_feeDenominator() = 10000`) simplifies percentage calculations while ensuring we never exceed 100%.
3. **`listTicketForSale(ticketId, price)`**
    - Only the ticket’s owner can do this (checked via the `isTicketOwner` modifier).
    - The function sets a new price, marks `forSale = true`, and the NFT can then be purchased by calling `buyTicket`.
4. **Event Updaters** (`setEventName`, `setEventLocation`, etc.)
    - Only the original event organizer can modify the event’s name, location, or schedule.
    - We do not store large data (like images) on-chain; instead, the `tokenURI` or external references cover that. This approach helps keep gas costs lower.
5. **Admin Controls**
    - **`pause()` / `unpause()`**: We inherited `Pausable`, allowing the contract’s owner to freeze certain functions (like `buyTicket`). This is essential for safety if we discover any bugs or suspicious activity.
    - **`setPlatformFee` / `setSecondaryRoyalty`**: The owner can adjust fees, but only up to a combined maximum of 100%. This helps strike a balance between fair revenue (for the platform and organizer) and user-friendliness.
    - **`changeTreasury(...)`**: We can update the treasury address if organizational needs change.

#### TCoin Contract

**Purpose & Design Rationale**  
We introduced `TCoin` as a dedicated ERC20 token, as requested in the project specifications.

**Interfaces and Inheritance**
- **`ERC20`** – The standard interface for fungible tokens, letting wallets easily store or transfer TCOIN.
- **`ERC20Burnable`** – Allows token holders to burn their tokens (reducing total supply if they wish).
- **`Ownable`** – Restricts minting new tokens, withdrawing ETH, etc., to the contract’s owner.

**Core Features**
1. **Conversion Rate**:
    - A fixed `RATE` of 10,000 means 1 ETH = 10,000 TCOIN.
    - This is a simple approach to keep the token’s value predictable.
2. **Minimum Purchase**:
    - Set at `0.001 ETH` to avoid spammy micro-purchases.
    - If someone tries to buy TCOIN with less than this threshold, the contract reverts.
3. **`buy()`**:
    - A user calls this with some Ether (`msg.value`).
    - As long as they meet the minimum purchase, we mint `msg.value * RATE` TCOIN to them.
4. **`sell(tokenAmount)`**:
    - Users can burn TCOIN in exchange for Ether at the same rate, provided the contract holds enough Ether.
    - The reason we do this is to allow an easy off-ramp if they don’t want leftover TCOIN.
    - `Ownable` can always inject more ETH or withdraw it if there’s a surplus.
5. **`withdraw(amount)`**:
    - Only the owner can remove Ether from the TCOIN contract’s balance. This design ensures we can reclaim contract liquidity if needed, or add more if the users need to sell TCOIN back.

We find this approach beneficial because **users aren’t forced** to pay with raw Ether. Instead, they **buy TCOIN** and can later resell it, effectively abstracting the event tickets from Ether price volatility. Meanwhile, the **contract logic** remains simpler, as we only handle a single ERC20 currency in `TicketMaster` (instead of multiple tokens or direct Ether payments).

### Steps to Deploy the Contract:

For our deployment pipeline, we chose to use the HardHat tooling.

1. Clone repository and install dependencies
```bash
git clone https://github.com/CSCD71/d-ticketmaster-chadurphy.git
cd d-ticketmaster-chadurphy
npm i
```

2. Navigate to `SmartContracts/hardhat.config.json` and change `networks.buildbear.accounts[0]` to a valid wallet private key you own.

3. Edit `SmartContracts/ignition/params.json` to be the correct params for the TicketMaster contract. For example, editing the default name, ERC721 token symbol, treasudy address or platformFeeBps. 

4. Run `npm run deploy`. Look in package.json to see exactly what happens, but it does the following:
- Compiles smart contracts
- Deploys both contracts using the TicketMasterModule in `ignition/modules` and the params defined in `ignition/params.json`. To change contract
- Alongside deployment, verifies the contracts by sending the compiled code to EtherScan.

### Steps to Test the Contract:
6. From project root, run `cd SmartContracts`
7. Run `npm run test`
8. For coverage, run `npm run coverage`

These commands run `npx hardhat test` and `npx hardhat coverage`, respectively.

Our coverage results are as follows:
![[Screenshot 2025-03-10 at 4.56.59 PM.png]]

**Coverage Strategies (In addition to testing expected behaviour of every function)**
- **Revert Cases / Negative Tests:**  
    To ensure **branch coverage**, we tested scenarios where the contract should fail (e.g., requiring a ticket to be for sale, preventing non-owners from calling certain functions, or setting an invalid fee). Every `require(...)` or `modifier` check (like `onlyOwner`) has a corresponding negative test to confirm it reverts as expected. This ensures that both “happy path” and “unhappy path” branches are covered.
- **Admin-Only Functions:**  
    We explicitly tested attempts by non-owners to call admin functions like `pause()`, `setPlatformFee()`, and `changeTreasury()`. Each test ensures the transaction reverts with the correct custom error. This approach covers all those “if not owner, revert” branches.
- **Pause/Unpause Logic:**  
    We wrote separate tests where `buyTicket()` is called under normal conditions vs. while paused. Attempting a purchase during the paused state hits the `whenNotPaused` branch, ensuring coverage there too.
- **Edge Cases:**
    - **Fee Limits:** We tested setting a fee above `10000` (i.e., more than 100%), which reverts.
    - **Royalty + Platform Fee** sum exceeding 100% is also tested, guaranteeing the combined coverage for “valid” vs. “invalid” input checks.
    - **Allowances & Transfer** in TCoin are tested with both correct allowance and insufficient allowance (triggering revert).
- **Full Lifecycle Scenarios:**
    - Creating an event, listing tickets, buying a ticket (primary sale), then relisting it (secondary sale) with different fees.
    - We also included checks on `tokenURI()` calls, `ERC721Enumerable` features (e.g., `totalSupply()`, `tokenByIndex()`), and the TCoin’s buy/sell/wallet interactions for complete coverage.

### Challenges Faced and Resolutions

#### Contract Verification

We found verifying contracts particularly difficult. Our initial deployment pipeline was using the Remix IDE, which worked for deployment, but the Sourcify plugin does not support verifying contracts custom buildbear networks. This forced us to move to using the hardhat tooling for deployment, which worked for verifying the standalone ERC20 TCoin contract, but not for verifying the ERC721 TickerMaster contract as it has the ERC20 contract as a dependency. Finally we moved to using EtherScan via the HardHat tooling and verifying on deployment, which allowed us to deploy and verify both contracts.

#### Realtime Updates
We spent significant time building realtime updates into the frontend app where possible. We were able to do this for all client-side operations (e.g the current user buying a ticket), but were not able to do so for actions triggered by users on another computer (e.g another user buying a ticket). This is because BuildBear's RPC doesn't support contract event integration on the free plan (we tried both with websocket RPC and with polling), so there was nothing we could do to overcome this. A user will be able to see the results of other's actions, but only by reloading the page. We cover in the 'improvements' section what we may change next time to counteract this.

#### Test Coverage for Edge Cases
We found that reaching 95% branch coverage was particularly difficult, as this meant we had to thoroughly test “unhappy paths” (e.g., buying one’s own ticket, insufficient allowance, zero address treasury, fees above 100%). Missing just a single revert branch in a function could lower coverage. To resolve this, we reviewed each require statement, modifier, and if/else branch, writing a direct test for each. We wish that Hardhat had better tooling to show us exactly what wasn't covered, to make our lives easier in this regard.

### Future Improvements and Additional Features

As we worked through this system, we’ve identified several improvements we might implement later:

1. **Dynamic Pricing**  
    Instead of ticket prices being statically set by the event organizer/secondary seller, we could consider auctions or dynamic seat pricing based on demand or time left until the event.
2. **Refund / Cancellation Process**  
    In case an event is canceled, we might want an automated way to return TCOIN or Ether to the buyers.
3. **Time-Based Pausing**  
    The contract could automatically pause ticket purchases once an event’s start time has passed, preventing post-event scalping, which is when people purchase event tickets after completion and resell them at a higher price.
4. **Bulk Minting**  
    For large events, we might add specialized bulk-mint functions to reduce overall gas costs.
5. **Realtime updates**
    We have done our best to implement realtime updates where possible, for example by refetching relevant data after a blockchain transaction has occurred (e.g refetch balance after buying a ticket).

    We actually did attempt to go one step further and implement realtime updates by listening on contract events, so that transactions performed by other users of the platform would be reflected immediately without a page refresh necessary. However, we ran into an issue that the BuildBear free plan does not seem to support event streaming, which we attempted through both websockets and also polling. This makes it impossible to detect other users actions in realtime.

    In a future iteration, we would use an Testnet RPC provider that supports realtime updates (preferably via a websocket RPC), which would then enable the above features.



