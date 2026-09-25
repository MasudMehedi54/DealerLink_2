# DealerLink – B2B Shop ↔ Dealer Platform (JavaFX)

Shop owners post product requests, dealers compete with private bids, the shop accepts the
best offer, and the dealer updates delivery. JavaFX 21 + SQLite + **Jackson**.

## Run

```bash
mvn clean javafx:run          # run from the project root (so data/ is found)
mvn clean package             # builds a runnable fat jar in target/
```

Demo logins: `shop1 / 1234`, `dealer1 / 1234`, `dealer2 / 1234`

> Tip: if you already have an old `dealerlink.db` from the prototype, delete it once so the
> new JSON seed data (7 products, low-stock examples) gets loaded cleanly.

## Project structure

```
DealerLink/
├── pom.xml                              ← org.json removed, jackson-databind added
├── data/
│   └── dealerlink-data.json             ← NEW external JSON file (settings + seed data)
├── docs/screenshots/                    ← how the new theme looks
└── src/main/
    ├── java/com/dealerlink/
    │   ├── Main.java
    │   ├── db/DatabaseManager.java      ← seeds DB from JSON (no more hard-coded SQL/ids)
    │   ├── json/                        ← NEW package – everything Jackson
    │   │   ├── JsonDataService.java     ← shared ObjectMapper: load(), settings(), exportToFile()
    │   │   ├── SeedData.java            ← Java mirror of the whole JSON document
    │   │   ├── AppSettings.java         ← "settings" block (with safe defaults)
    │   │   ├── InventorySeed.java       ← inventory row (dealer by username, product by name)
    │   │   └── WeatherResponse.java     ← typed mapping of the Open-Meteo REST response
    │   ├── dao/  model/  service/  ui/
    └── resources/css/theme.css          ← fully redesigned
```

## 1. JSON with Jackson – what was added

**External file `data/dealerlink-data.json`** has four blocks:

| Block       | Maps to               | Used for |
|-------------|-----------------------|----------|
| `settings`  | `AppSettings`         | app name, currency symbol, low-stock threshold, polling intervals, weather API URL |
| `users`     | `List<User>`          | demo shop/dealer accounts |
| `products`  | `List<Product>`       | product catalog |
| `inventory` | `List<InventorySeed>` | dealer stock (refers to `dealerUsername` + `productName`, not ids) |
| `cities`    | `List<CityCoordinate>`| offline city → latitude/longitude table for weather |

How it flows:

1. `Main` → `DatabaseManager.initializeDatabase()` → `JsonDataService.load()`
   → `ObjectMapper.readValue(file, SeedData.class)`.
2. Each user/product/inventory row is inserted **only if it isn't already in the DB**. So you can
   add a new product to the JSON, restart, and it appears — without wiping orders or stock the
   dealer edited in the app.
3. `settings` are read live: `DealerDashboard` uses `lowStockThreshold`, `ShopDashboard` uses
   `orderMonitorIntervalSeconds`, `UiUtils.currency()` uses `currencySymbol`.
4. `WeatherService` now parses the REST response with Jackson into `WeatherResponse`
   (replacing `org.json`) and shows a readable condition (e.g. "Thunderstorm – deliveries may be delayed").
5. **Export JSON** buttons (Shop → Orders & Delivery, Dealer → Orders & Deliveries) write the
   table rows to a `.json` file with `ObjectMapper.writeValue(...)`.

6. **Weather + geocoding**: accounts created on the Register screen have no coordinates (0,0).
   `GeocodingService` now resolves the typed city automatically — first from the offline
   `cities` table in the JSON file (26 Bangladesh cities, works without internet), then from the
   Open-Meteo geocoding API (parsed into `GeocodingResponse` with Jackson) — and saves the result
   to the `users` table. "Check Area Weather" shows the weather at **both** the shop and the
   selected dealer's location.

Use a different file: `-Ddealerlink.data=C:\path\to\file.json`.
If the file is missing, the app still starts with built-in default settings (and logs a warning).

## 2. CSS redesign – `theme.css`

- **Design tokens**: every color is defined once on `.root` (`-dl-primary`, `-dl-success`, …).
  Re-brand the whole app by editing those lines.
- **Login/Register**: indigo → violet gradient with soft glow spots and a floating white card.
- **Dashboards**: gradient header, pill-shaped tabs, white rounded content card.
- **Inputs**: focus glow ring, hover border, red error state.
- **Buttons**: gradient primary with hover lift + shadow, outline secondary, translucent header button.
- **Tables**: zebra rows, hover highlight, left accent bar on the selected row; green bar for best
  price, red bar for low stock; columns now stretch to the full width (no truncated headers).
- **Badges** with borders and distinct colors per status; thin rounded scroll bars; dark tooltips.
- All inline `setStyle(...)` calls in Java were moved into CSS classes (`auth-root`,
  `dashboard-root`, `button-header`, `currency-cell`, `app-title-small`).

## 3. Bugs fixed while reconstructing

| Problem | Fix |
|---|---|
| `getGeneratedKeys()` throws `SQLFeatureNotSupportedException` on newer sqlite-jdbc builds → creating requests/bids/orders could crash | `DatabaseManager.lastInsertId()` using `SELECT last_insert_rowid()` |
| Background threads were non-daemon → closing the window without Logout kept the JVM running | daemon thread factory in `BackgroundTaskService` |
| Dealer's inventory-watch executor was never shut down → every login leaked a thread | now scheduled on `bgService`, which Logout shuts down |
| Shop could accept a quotation twice for the same request → duplicate orders | guard: request must still be `OPEN`; table refreshes after accept |
| `WeatherService` set the interrupt flag on normal I/O errors; accounts with lat/lon 0,0 queried the ocean | separated catch blocks; clear message when coordinates are missing |

## 4. Suggested next modifications

1. **Hash passwords** (`UserDAO.register/login`) – store a BCrypt/SHA-256+salt hash, never plain text
   (the JSON demo passwords would then be hashed on seeding).
2. **Keep order and delivery status in sync** (`DealerDashboard` orders tab) – setting delivery to
   `DELIVERED` should also set the order to `DELIVERED` (one DAO method in a transaction).
3. **Wrap multi-step writes in a transaction** (`ShopDashboard` accept button: accept quote +
   reject others + update request + place order) so a failure can't leave half-updated data.
4. **Move DB calls off the UI thread** – `quotesCol`/`yourBidCol` run one SQL query per row while
   rendering; load counts in one `GROUP BY` query inside a JavaFX `Task`.
5. **Dashboard KPI cards** – a row above the tabs (e.g. "Open requests 3 · Best saving ৳25 ·
   Low-stock items 1"); the CSS tokens make these easy to style.
6. **Import from JSON** – a dealer "Import stock (.json)" button re-using `JsonDataService.mapper()`
   to read a list of `InventorySeed`.
7. **Split big UI classes** – `ShopDashboard`/`DealerDashboard` build every tab inline; one class per
   tab (or FXML + controller) makes them easier to test and change.
8. **Add `module-info.java`** if you package with `jlink`/`jpackage`
   (`requires javafx.controls; requires java.sql; requires java.net.http; requires com.fasterxml.jackson.databind; opens com.dealerlink.model to javafx.base, com.fasterxml.jackson.databind; opens com.dealerlink.json to com.fasterxml.jackson.databind;`).
