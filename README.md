# TindaTrack

TindaTrack is a single-page point-of-sale and sari-sari store management app. The ASP.NET Core API serves the frontend and persists products, checkout sales, audit logs, and admin actions in SQL Server.

## Standalone demo

Double-click the project-folder `index.html` to open the site in a browser. When opened this way, it runs in standalone demo mode: it uses the sample catalog, supports ordering and the demo admin screens, and saves changes in that browser's local storage. When creating inventory products, you can choose from 41 common emojis or enter a custom one, add an optional PNG, JPG, or WebP image (up to 2 MB), and remove products that have not been used in sales. Products in sales history are protected from removal. The demo admin PIN is `1234`. This mode does not connect to SQL Server and should not be used for real sales.

For SQL-backed use, do not open `index.html` directly. Follow the setup steps below and use the URL printed by `dotnet run`.

## Use Azure SQL and publish online

Azure SQL hosts the database; it does not host the website or API. To let other devices use the same data, publish this ASP.NET Core app (which serves the frontend and `/api`) to Azure App Service and connect it to an Azure SQL Database.

1. In the Azure portal, create an **Azure SQL Database** named `TindaTrackDB` and its logical SQL server. Configure a database administrator or Microsoft Entra administrator.
2. In the SQL server's **Networking** settings, allow connections from your API host. For local setup/testing, add only your current client IP temporarily. Do not allow unrestricted public access.
3. In SSMS, connect to `<your-server>.database.windows.net`, port `1433`, and select the Azure database. Azure SQL Database does not allow the `CREATE DATABASE`/`USE` preamble in `database/schema.sql`: create the database in the portal, then run the schema statements starting at `IF OBJECT_ID(...)` with `TindaTrackDB` selected. To update an existing database, rerun those guarded table/column statements so the product image column is added.
4. Publish the app:

   ```powershell
   dotnet publish .\backend\TindaTrack.Api.csproj --configuration Release --output .\publish
   ```

   Deploy the contents of `.\publish` to an Azure App Service configured for the project's .NET runtime. The publish output includes the frontend files.
5. In the App Service's **Environment variables** (or **Configuration**) settings, add `ConnectionStrings__TindaTrack` with the Azure SQL connection string. Use the server, database, and SQL login details from your Azure setup:

   Use the Azure SQL connection string with encryption enabled. Store it in the App Service settings or a Key Vault reference; never commit credentials to `appsettings.json` or source control. Restart the App Service after changing settings.
6. Open the App Service's HTTPS URL from your devices. Run a test sale and confirm the sale and changed stock appear in SSMS connected to the Azure server.

For a local API test against Azure SQL, set the same `ConnectionStrings__TindaTrack` environment variable in the terminal before running `dotnet run`. Environment configuration overrides the LocalDB value in `appsettings.json`.

## Run locally in Visual Studio Code

1. Install the .NET 10 SDK and SQL Server. The default connection string targets `(localdb)\MSSQLLocalDB`.
2. Open `database/schema.sql` in SQL Server Management Studio and execute it to create `TindaTrackDB` and the initial catalog. To update an existing database with the product image column, execute the script again against that database; its table and column checks are safe to rerun.
3. Check `backend/appsettings.json` and make sure its `TindaTrack` connection string points to the same SQL Server instance where you ran the schema. For SQL Server Express, for example, use `Server=localhost\\SQLEXPRESS`.
4. In the VS Code terminal, from the project folder, run:

   ```powershell
   dotnet run --project .\backend\TindaTrack.Api.csproj
   ```

5. Open the local URL printed by `dotnet run`. The demo administrator PIN is `1234`.

The demo PIN is intentionally convenient for local coursework only. Set a unique administrator PIN before deploying the app for real store use. Admin API requests require the short-lived, HTTP-only login cookie; the PIN endpoint is rate-limited.

GCash is recorded as a payment method after the cashier confirms receipt; this project does not connect to or verify an external GCash payment.

## Included workflows

- Landing page navigation telemetry and live clock
- Product search, category filters, stock-limited cart, cash/GCash checkout, and receipt
- PIN-protected admin dashboard, inventory updates, product creation, and sales cancellation with stock restoration
- Sales trend and top-product analytics

## Event-driven C# concepts

The API demonstrates concepts from the event-driven C# handouts without changing the frontend workflows:

- **Delegates and events (Handout 1):** typed `EventHandler<TEventArgs>` events are published after a sale commits or an inventory change succeeds.
- **Collections and generics (Handout 2):** checkout uses `List<T>` and `Dictionary<TKey, TValue>` to merge repeated cart lines while preserving their original order; the category allow-list uses `HashSet<T>`.
- **Exception handling (Handout 3):** API validation returns clear client errors, while database and overflow exceptions are handled explicitly and logged where appropriate. Checkout and sale cancellation use SQL transactions so failures do not leave partial stock or sales changes.
- **Threading and race conditions (Handout 4):** a hosted background worker consumes domain-event notifications through a bounded, thread-safe `Channel<T>`. Checkout also relies on SQL transactions and row locks to prevent concurrent checkouts from overselling stock. Raw `Thread.Abort` and `Thread.Sleep` are intentionally not used in the web API.

The event worker records operational messages in the API log; it does not change POS responses or add user-facing behavior.

### Where to find the handout examples

Search the backend for the `Handout 1`, `Handout 2`, `Handout 3`, or `Handout 4` comments:

- `backend/StoreEventProcessor.cs`: Handout 1 event-argument types, typed events, and event subscribers; Handout 4 thread-safe channel and background consumer.
- `backend/controller/PosController.cs`, `Checkout`: Handout 2 cart `List<T>`/`Dictionary<TKey, TValue>`; Handout 4 stock transaction/row locks; Handout 1 sale-completed event; Handout 3 overflow handling.
- `backend/controller/AdminController.cs`, `AddProduct`, `AdjustStock`, `RemoveProduct`, and `CancelTransaction`: Handout 1 inventory events; Handout 3 atomic sale-cancellation transaction.
