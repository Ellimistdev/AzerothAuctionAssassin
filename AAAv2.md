# Azeroth Auction Assassin Migration Plan

After reviewing the codebase for Azeroth Auction Assassin (AAA) and Azeroth Auction Target (AAT), I see an opportunity to refactor this project by separating concerns to improve maintainability, performance, and portability. Here's a comprehensive migration plan:

## Current Architecture Assessment

The current architecture has several limitations:

1. **Tightly coupled UI and business logic**: Most functionality is embedded directly in UI classes.
2. **Duplicate code**: Similar functionality exists across both applications.
3. **Limited testing capability**: Business logic intertwined with UI makes unit testing difficult.
4. **Poor separation of concerns**: Data fetching, processing, and presentation are mixed together.
5. **Limited portability**: Difficult to reuse code in different UI frameworks or environments.

## Migration Strategy: Core Library + UI Frontends

### Phase 1: Create Shared Core Library

Create a Python package called `azeroth_auction_core` with these modules:

1. **API Module**
   - Move API-related functionality from `api_requests.py`
   - Create classes for different API endpoints (Blizzard, SaddleBag)
   - Implement proper error handling and retry logic
   - Add caching to reduce API calls

2. **Data Models**
   - Define data models for auctions, items, realms, etc.
   - Add validation and serialization/deserialization

3. **Business Logic**
   - Auction analysis logic from `mega_alerts.py`
   - Item matching algorithms
   - Price analysis functions

4. **Configuration Management**
   - Settings storage and retrieval
   - User preference persistence
   - Environment configuration

5. **Notification System**
   - Discord webhook integration
   - Alert generation
   - Formatting utilities

### Phase 2: Decouple UI from Business Logic

1. **Refactor AAA Application**
   - Remove direct API calls and replace with core library
   - Implement UI-specific state management
   - Use MVC or MVVM pattern for UI structure

2. **Refactor AAT Application**
   - Similar refactoring to AAA
   - Share common UI components where applicable

3. **Common UI Components**
   - Create shared UI components used by both apps
   - Implement consistent styling and user experience

### Phase 3: Improve Core Features and Performance

1. **Performance Optimizations**
   - Implement asynchronous processing using `asyncio`
   - Add parallel execution for data processing
   - Optimize memory usage with generators and lazy loading

2. **Extensibility Framework**
   - Create plugin architecture for custom analyzers
   - Add hooks for additional data sources
   - Enhance support for user-defined alert criteria

## Implementation Details

### Core Library Structure

```
azeroth_auction_core/
├── __init__.py
├── api/
│   ├── __init__.py
│   ├── blizzard.py
│   ├── saddlebag.py
│   └── cache.py
├── models/
│   ├── __init__.py
│   ├── auction.py
│   ├── item.py
│   ├── pet.py
│   └── realm.py
├── analysis/
│   ├── __init__.py
│   ├── matcher.py
│   ├── pricing.py
│   └── statistics.py
├── config/
│   ├── __init__.py
│   ├── settings.py
│   └── persistence.py
├── notifications/
│   ├── __init__.py
│   ├── discord.py
│   └── formatters.py
└── utils/
    ├── __init__.py
    ├── async_helpers.py
    └── data_processing.py
```

### Example Code: API Client Refactoring

```python
# Current code in api_requests.py
@retry(stop=stop_after_attempt(3))
def get_listings_single(connectedRealmId: int, access_token: str, region: str):
    print(f"gather data from connectedRealmId {connectedRealmId} of region {region}")
    if region == "NA":
        url = f"https://us.api.blizzard.com/data/wow/connected-realm/{str(connectedRealmId)}/auctions?namespace=dynamic-us&locale=en_US"
    elif region == "EU":
        url = f"https://eu.api.blizzard.com/data/wow/connected-realm/{str(connectedRealmId)}/auctions?namespace=dynamic-eu&locale=en_EU"
    else:
        print(f"{region} is not yet supported, reach out for us to add this region option")
        exit(1)

    headers = {"Authorization": f"Bearer {access_token}"}
    req = requests.get(url, headers=headers, timeout=20)
    auction_info = req.json()
    return auction_info["auctions"]

# Refactored code in azeroth_auction_core/api/blizzard.py
class BlizzardAPI:
    def __init__(self, client_id, client_secret, region="EU", cache_ttl=3600):
        self.client_id = client_id
        self.client_secret = client_secret
        self.region = region
        self.cache = APICache(ttl=cache_ttl)
        self._token = None
        self._token_expiry = 0
        
    async def get_auth_token(self):
        """Get an OAuth token with proper caching and expiry handling"""
        current_time = int(time.time())
        if self._token and current_time < self._token_expiry:
            return self._token
            
        async with aiohttp.ClientSession() as session:
            async with session.post(
                "https://oauth.battle.net/token",
                data={"grant_type": "client_credentials"},
                auth=aiohttp.BasicAuth(self.client_id, self.client_secret)
            ) as response:
                if response.status != 200:
                    raise APIError(f"Failed to get token: {await response.text()}")
                data = await response.json()
                self._token = data["access_token"]
                self._token_expiry = current_time + data["expires_in"] - 300  # 5 min buffer
                return self._token
                
    async def get_auctions(self, connected_realm_id):
        """Get auction data for a realm with caching"""
        cache_key = f"auctions:{self.region}:{connected_realm_id}"
        cached_data = self.cache.get(cache_key)
        if cached_data:
            return cached_data
            
        base_url = "https://us.api.blizzard.com" if self.region == "NA" else "https://eu.api.blizzard.com"
        namespace = f"dynamic-{self.region.lower()}"
        locale = "en_US" if self.region == "NA" else "en_EU"
        
        url = f"{base_url}/data/wow/connected-realm/{connected_realm_id}/auctions"
        token = await self.get_auth_token()
        
        async with aiohttp.ClientSession() as session:
            async with session.get(
                url,
                params={"namespace": namespace, "locale": locale},
                headers={"Authorization": f"Bearer {token}"},
                timeout=20
            ) as response:
                if response.status != 200:
                    raise APIError(f"Failed to get auctions: {await response.text()}")
                    
                data = await response.json()
                auctions = data.get("auctions", [])
                self.cache.set(cache_key, auctions)
                return auctions
```

### Example Code: Alert Processing

```python
# Current code in mega_alerts.py
def clean_listing_data(auctions, connected_id):
    all_ah_buyouts = {}
    all_ah_bids = {}
    pet_ah_buyouts = {}
    pet_ah_bids = {}
    ilvl_ah_buyouts = []
    pet_ilvl_ah_buyouts = []

    if len(auctions) == 0:
        print(f"no listings found on {connected_id} of {mega_data.REGION}")
        return

    def add_price_to_dict(price, item_id, price_dict, is_pet=False):
        if is_pet:
            if price < mega_data.DESIRED_PETS[item_id] * 10000:
                if item_id not in price_dict:
                    price_dict[item_id] = [price / 10000]
                elif price / 10000 not in price_dict[item_id]:
                    price_dict[item_id].append(price / 10000)
        elif price < mega_data.DESIRED_ITEMS[item_id] * 10000:
            if item_id not in price_dict:
                price_dict[item_id] = [price / 10000]
            elif price / 10000 not in price_dict[item_id]:
                price_dict[item_id].append(price / 10000)

    # Much more code here...

# Refactored code in azeroth_auction_core/analysis/matcher.py
class AuctionMatcher:
    def __init__(self, config):
        self.config = config
        
    async def find_matching_auctions(self, auctions, realm_id):
        """Find auctions matching user criteria"""
        if not auctions:
            logger.info(f"No listings found on {realm_id} of {self.config.region}")
            return None
            
        results = {
            'items': {},
            'pets': {},
            'ilvl_items': [],
            'pet_levels': []
        }
        
        # Process auctions in parallel chunks for better performance
        chunk_size = 1000
        auction_chunks = [auctions[i:i+chunk_size] for i in range(0, len(auctions), chunk_size)]
        
        tasks = [self._process_auction_chunk(chunk, results) for chunk in auction_chunks]
        await asyncio.gather(*tasks)
        
        if all(not v for v in results.values()):
            logger.info(f"No matching listings found on {realm_id}")
            return None
            
        return results
        
    async def _process_auction_chunk(self, auctions, results):
        """Process a chunk of auctions"""
        for auction in auctions:
            await self._process_auction(auction, results)
            
    async def _process_auction(self, auction, results):
        """Process a single auction"""
        item_id = auction.get("item", {}).get("id")
        if not item_id:
            return
            
        # Handle regular items
        if item_id in self.config.desired_items and item_id != 82800:
            await self._handle_regular_item(auction, item_id, results)
            
        # Handle battle pets
        elif item_id == 82800:
            pet_id = auction.get("item", {}).get("pet_species_id")
            if pet_id:
                if pet_id in self.config.desired_pets:
                    await self._handle_pet(auction, pet_id, results)
                    
                if pet_id in [p["petID"] for p in self.config.desired_pet_levels]:
                    await self._handle_pet_level(auction, pet_id, results)
                    
        # Handle ilvl items
        if self.config.desired_ilvl_items and item_id in self.config.desired_ilvl_items.get("item_ids", []):
            await self._handle_ilvl_item(auction, results)
```

### Example Code: UI Implementation

```python
# Python frontend using azeroth_auction_core
from azeroth_auction_core.api.blizzard import BlizzardAPI
from azeroth_auction_core.config.settings import ConfigManager
from azeroth_auction_core.analysis.matcher import AuctionMatcher
from azeroth_auction_core.notifications.discord import DiscordNotifier

class AuctionSniperApp:
    def __init__(self):
        self.config = ConfigManager()
        self.blizzard_api = BlizzardAPI(
            self.config.get("blizzard.client_id"),
            self.config.get("blizzard.client_secret"),
            self.config.get("blizzard.region")
        )
        self.matcher = AuctionMatcher(self.config)
        self.notifier = DiscordNotifier(self.config.get("discord.webhook_url"))
        
    async def run_scan(self):
        """Run a full scan of all realms"""
        realms = await self.blizzard_api.get_connected_realms()
        
        for realm in realms:
            auctions = await self.blizzard_api.get_auctions(realm["id"])
            matches = await self.matcher.find_matching_auctions(auctions, realm["id"])
            
            if matches:
                await self.notifier.send_alerts(matches, realm)
```

## Benefits of Migration

1. **Improved Code Organization**
   - Clear separation of responsibilities
   - Better organization around domains

2. **Enhanced Performance**
   - Asynchronous processing reduces wait times
   - Proper caching reduces API calls

3. **Better Developer Experience**
   - Clearer API for working with auction data
   - Reduced coupling makes changes safer
   - Improved testability

4. **Greater Portability**
   - Core library usable in various UI frameworks (PyQt, Tkinter, web)
   - Possible to build alternative UIs or command-line tools

5. **Future-Proofing**
   - Easier to adapt to Blizzard API changes
   - More straightforward to add new features

## Timeline Estimate

- **Phase 1 (4-6 weeks)**: Core library development and testing
- **Phase 2 (3-4 weeks)**: UI refactoring for AAA and AAT
- **Phase 3 (3-4 weeks)**: Performance optimizations and enhancements

## Conclusion

This migration plan will transform the current tightly coupled applications into a more modular, maintainable system. By separating core business logic from UI concerns, we'll create a more robust foundation that's easier to extend, maintain, and adapt to future changes.