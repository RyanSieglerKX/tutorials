# Building a Market Data API with KURL and REST in KDB-X

## The Journey from Cloud Storage to Production API

In this tutorial, we'll build a complete data pipeline:
1. 🌐 Fetch data from public APIs using **KURL**
2. ☁️ Upload market data to Google Cloud Storage
3. 📥 Download and parse CSV data into KDB-X tables
4. 🚀 Expose analytics as REST APIs
5. 🧪 Test our APIs with real requests

**What you'll learn:**
- How to use KURL for HTTP requests (public and authenticated)
- Working with GCP buckets using OAuth2 authentication
- Creating RESTful APIs that expose KDB-X functions

**See the docs:**
- [kurl](https://docs.kx.com/latest/kdb-x/integrations/kurl.htm)
- [REST](https://docs.kx.com/latest/kdb-x/integrations/rest.htm)

**Prerequisites:**
- KDB-X installed - https://developer.kx.com/products/kdb-x/install
- Download the [OHLC.csv file](https://github.com/KxSystems/tutorials/blob/main/KDB-X/Modules/ai-libs/src/OHLC.csv) (stock market data) to your working directory
- GCP account with authentication configured and a cloud bucket created

Let's begin! 🎯

---
## Part 0: Start KDB-X
**After installing KDB-X, in your terminal, run:**
```bash
q
```

## Part 1: Introduction to KURL - Public APIs

KURL is KDB-X's HTTP client library. Let's start simple by fetching data from a **public API** that requires no authentication.

We'll query the US Census Bureau's API to get population data by state.

```q
// Load the KURL module
.kurl:use`kx.kurl

// Query US Census API - no authentication needed!
censusURL:"https://api.census.gov/data/2020/dec/pl?get=NAME,P1_001N&for=state:*";

// Make synchronous HTTP GET request
censusResp:.kurl.sync(censusURL; `GET; ::);

// Parse JSON response
censusData:.j.k last censusResp;

// Display first 5 states
show "First 5 states with population data:";
show 5#censusData;
```

### 💡 What Just Happened?

```q
.kurl.sync(url; method; options)
```

- **url**: The endpoint to call
- **method**: HTTP method (`` `GET``, `` `POST``, etc.)
- **options**: Headers, body, auth (`` `::` means none)

Returns: `(statusCode; responseBody)`

For public APIs, it's that simple! No authentication configuration needed.

---

## Part 2: Authenticated APIs - Google Cloud Storage

Now let's level up! We'll interact with **private** cloud storage using OAuth2 authentication.

### Setting Up GCP Authentication

First, we need to register our GCP credentials with KURL. This tells KURL to automatically add authentication headers to requests going to `*.googleapis.com`.

**Exit q session with ``\\`` and In your terminal, run:**
```bash
gcloud init
export GCP_TOKEN=$(gcloud auth print-access-token)
```

**Then reenter a q session with ``q``, and in q:**
```q
// load the kurl module
.kurl:use`kx.kurl

// Register OAuth2 credentials for Google Cloud
.kurl.register(`oauth2; "*.googleapis.com"; ""; enlist[`access_token]!enlist getenv`GCP_TOKEN)
```

---

## Part 3: Uploading Data to GCS

Let's upload our OHLC.csv file (stock market data) to a Google Cloud Storage bucket.

**The data:** Daily OHLC (Open, High, Low, Close) prices for 8 stocks from January to July 2025.

**Remember to update your GCS bucket name below!**

```q
// Define bucket and object names
bucket:"insights-core-bucket";  // Change to your bucket name
object:"OHLC.csv";

// Construct upload URL
url:"https://storage.googleapis.com/upload/storage/v1/b/",bucket,"/o?uploadType=media&name=",object;

// Set content type header
headers:enlist["Content-Type"]!enlist "text/csv";

// Read local CSV file as bytes
csvBytes:read1 `:OHLC.csv;

// Upload to GCS using POST
resp:.kurl.sync(url; `POST; `headers`body!(headers; csvBytes));

// Check for success (HTTP 200)
if[200<>first resp; 'last resp];
```

---

## Part 4: Listing Files in the Bucket

Let's verify our upload by listing files in the bucket.

```q
// List objects in bucket (max 10 results)
listURL:"https://storage.googleapis.com/storage/v1/b/",bucket,"/o?maxResults=10";
r:.kurl.sync(listURL; `GET; ::);

// Check response status
if[200<>first r; 'last r];

// Parse JSON and extract file names
objects:.j.k last r;
fileList:objects[`items][;`name];

show "Files in bucket:";
show fileList;
```

---

## Part 5: Getting File Metadata

Before downloading, let's inspect the file's properties.

```q
// Get metadata for our uploaded file
fileURL:"https://storage.googleapis.com/storage/v1/b/",bucket,"/o/",object;
fileInfo:.kurl.sync(fileURL; `GET; ::);

// Parse metadata
metadata:.j.k last fileInfo;

// Display file information
show "File size: ",$[10h=type metadata`size; metadata`size; string metadata`size]," bytes";
show "Content type: ",metadata`contentType;
show "Created: ",metadata`timeCreated;
```

---

## Part 6: Downloading and Parsing Data

Now for the magic moment - let's download our CSV and transform it into a KDB-X table!

### 📥 Download from GCS

```q
// Download URL with alt=media to get file content
downloadURL:"https://storage.googleapis.com/storage/v1/b/",bucket,"/o/",object,"?alt=media";

// Download the file
resp:.kurl.sync(downloadURL; `GET; ::);
if[200<>first resp; 'last resp];

// Extract CSV content from response
csvData:last resp;

show "Size: ",string count csvData," characters";
```

### 🔄 Parse CSV to KDB-X Table

The CSV contains columns: date, sym, company, close, volume, open, high, low

We'll parse each column with the appropriate type.

```q
// Define column types:
// D = Date, S = Symbol, S = Symbol, F = Float (x5), J = Long
types:"DSSFFFFFJ";

// Parse CSV with column headers (enlist csv)
t:(types; enlist csv) 0: csvData;

show "✅ Data parsed into table 't'!";
show "";
show "First 5 rows:";
show 5#t;
```

---

## Part 7: Data Exploration

Let's explore what we have!

```q
// Basic statistics
show "Total records: ",string count t;
show "Unique symbols: ",string count distinct t`sym;
show "Date range: ",string[min t`date]," to ",string max t`date;
show "";

// Latest prices by symbol
show "Latest prices:";
show select last close by sym from t;
```

### 📈 Analytics Query

Now we can run powerful analytics on our data!

```q
// Calculate statistics by symbol
stats:select avgClose:avg close,maxClose:max close,minClose:min close,totalVol:sum volume by sym from t;

show stats;
```

---

## Part 8: Building a REST API Server

Now for the exciting part - let's expose our analytics as REST APIs!

### 🏗️ Setting Up the REST Framework

```q
// Load REST module
.com_kx_rest:use`kx.rest;
.rest:.com_kx_rest;  // Create convenient alias

// Initialize framework with autoBind
// autoBind:1b means REST will handle .z.ph and .z.pp
.rest.init enlist[`autoBind]!enlist[1b];

show "REST framework initialized!";
```

---

## Part 9: Define API Handler Functions

These functions will process API requests and return data.

**Important Note:** The REST framework automatically parses parameters based on their type specification, so `x[`arg;`sym]` is already a symbol - no casting needed!

```q
// Handler 1: Get latest price for a symbol
.api.getPrice:{[x]symbol:x[`arg;`sym]; select date:last date, close:last close, volume:last volume by sym from t where sym=symbol};

// Handler 2: Get all available symbols
.api.getAllSymbols:{[x]select distinct sym from t};

// Handler 3: Get latest prices for all symbols
.api.getLatestPrices:{[x]select date:last date, close:last close by sym from t};

// Handler 4: Get statistics for a symbol
.api.getStats:{[x]symbol:x[`arg;`sym];select minPrice:min close,maxPrice:max close,avgPrice:avg close, totalVolume:sum volume,tradeDays:count i by sym from t where sym=symbol};

// Handler 5: Get historical price data
.api.getPriceHistory:{[x]symbol:x[`arg;`sym];startDate:x[`arg;`startDate];endDate:x[`arg;`endDate]; select date, open, high, low, close, volume from t where sym=symbol, date within (startDate;endDate)};
```


---

## Part 10: Register API Endpoints

Now we map URL paths to our handler functions.

**Endpoint Registration Syntax:**
```q
.rest.register[
    method;        // `get, `post, etc.
    path;          // URL path like "/price"
    description;   // Human-readable description
    handler;       // Function to call
    params         // Parameter definitions
]
```

```q
// Health check endpoint - simple test
.rest.register[`get;"/hc";"Health check endpoint";{"ok"};()!()];

// GET /symbols - List all stock symbols
.rest.register[`get;"/symbols";"Returns list of all available symbols";.api.getAllSymbols;()!()];

// GET /latest - Latest prices for all stocks
.rest.register[`get;"/latest";"Returns latest prices for all symbols";.api.getLatestPrices;()!()];

// GET /price?sym=AXON - Latest price for specific symbol
.rest.register[`get;"/price";"Returns the latest price for a given symbol";.api.getPrice;.rest.reg.data[`sym;-11h;1b;`;"Stock symbol"]];

// GET /stats?sym=AXON - Statistics for symbol
.rest.register[`get;"/stats";"Returns price statistics for a given symbol";.api.getStats;.rest.reg.data[`sym;-11h;1b;`;"Stock symbol"]];

// GET /history?sym=AXON&startDate=2025.01.01&endDate=2025.03.31
.rest.register[`get;"/history";"Returns historical price data for a symbol within a date range";.api.getPriceHistory;.rest.reg.data[`sym;-11h;1b;`;"Stock symbol"],.rest.reg.data[`startDate;-14h;1b;.z.d-30;"Start date (YYYY.MM.DD)"],.rest.reg.data[`endDate;-14h;1b;.z.d;"End date (YYYY.MM.DD)"]];
```

### 📝 Understanding Type Codes

- `-11h` = Symbol atom (`` `AXON``)
- `-14h` = Date atom (`2025.01.01`)
- `-6h` = Integer atom
- `-9h` = Float atom

---

## Part 11: Start the Server

Now we open port 8080 to accept HTTP requests.

```q
// Start HTTP server on port 8080
\p 8080

show "🚀 REST API Server running on port 8080!";
show "";
show "Available endpoints:";
show "  GET /hc";
show "  GET /symbols";
show "  GET /latest";
show "  GET /price?sym=AXON";
show "  GET /stats?sym=AXON";
show "  GET /history?sym=AXON&startDate=2025.01.01&endDate=2025.03.31";
```

---

## Part 12: Testing Our API

Time to test our creation! 

**⚠️ Important:** Run these tests in a **separate q session** to avoid deadlock.

The server session can't call itself synchronously!

### Initialize KURL Client

In a new q session or terminal:

```q
// Load KURL in client session
.kurl:use`kx.kurl

```

### Test 1: Health Check ❤️

Verify the server is responding.

```q
show "Testing health check...";
resp:.kurl.sync("http://localhost:8080/hc"; `GET; ::);
show "Status: ",string first resp;
show "Response: ",last resp;
```

Expected output:
```
Testing health check...
Status: 200
Response: "ok"
```

### Test 2: Get All Symbols 📋

```q
show "Getting all symbols...";
resp:.kurl.sync("http://localhost:8080/symbols"; `GET; ::);

// Parse JSON response
symbols:.j.k last resp;
show "Available symbols:";
show symbols;
```

### Test 3: Get Latest Prices 💰

```q
show "Getting latest prices for all symbols...";
resp:.kurl.sync("http://localhost:8080/latest"; `GET; ::);

if[200=first resp;latest:.j.k last resp;show latest;];
```

### Test 4: Get Price for Specific Symbol 🎯

```q
show "Getting latest price for AXON...";
resp:.kurl.sync("http://localhost:8080/price?sym=AXON"; `GET; ::);

if[200=first resp;price:.j.k last resp;show price;];
```

Expected output:
```json
[{
  "sym": "AXON",
  "date": "2025.07.31",
  "close": 755.49,
  "volume": 513614
}]
```

### Test 5: Get Statistics 📊

```q
show "Getting statistics for AXON...";
resp:.kurl.sync("http://localhost:8080/stats?sym=AXON"; `GET; ::);

if[200=first resp;stats:.j.k last resp;show stats;];
```

### Test 6: Get Historical Data 📈

```q
show "Getting historical data for AXON (Jan-Mar 2025)...";
url:"http://localhost:8080/history?sym=AXON&startDate=2025.01.01&endDate=2025.03.31";
resp:.kurl.sync(url; `GET; ::);

if[200=first resp;history:.j.k last resp;show "First 5 records:";show 5#history;];
```

---

## Part 13: Testing with curl

You can also test from the command line!

```bash
# Health check
curl http://localhost:8080/hc

# Get all symbols
curl http://localhost:8080/symbols

# Get latest prices
curl http://localhost:8080/latest

# Get specific price
curl "http://localhost:8080/price?sym=AXON"

# Get statistics
curl "http://localhost:8080/stats?sym=CEG"

# Get historical data
curl "http://localhost:8080/history?sym=AXON&startDate=2025.01.01&endDate=2025.03.31"
```

---

## Part 14: Stop the REST server

To stop the REST server, from the new q session:

```q
h:hopen `:localhost:8080
h "exit 0"
```

---

## 🎉 Conclusion

Congratulations! You've built a complete data pipeline:

1. ✅ Used KURL to fetch data from public APIs
2. ✅ Authenticated with GCP using OAuth2
3. ✅ Uploaded data to cloud storage
4. ✅ Downloaded and parsed CSV into kdb-X tables
5. ✅ Built a RESTful API exposing analytics
6. ✅ Tested all endpoints with real requests


### 📚 Key Takeaways

**KURL:**
- Simple sync/async HTTP client
- Supports OAuth2, AWS, Azure authentication
- Works with any HTTP API

**REST Module:**
- Declarative endpoint registration
- Automatic parameter parsing and validation
- Type-safe with q type specifications
- Returns JSON by default

Happy coding! 🎯
