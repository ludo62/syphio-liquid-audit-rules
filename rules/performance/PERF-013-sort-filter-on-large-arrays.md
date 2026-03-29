PERF-013 | | sort on collections.all.productsCategory: PerformanceSeverity: 🔴 CRITICALAffects: Any template applying | sort or | sort_natural to the global product catalog or large collections — typically found in search results, collection filters, or custom "all products" displays.Added: 2026-03 · Last updated: 2026-03ContextThe | sort filter is a blocking, memory-intensive operation with a time complexity of $O(n \log n)$. When applied to Liquid arrays, it requires the entire input set to be materialized in memory before the first comparison occurs. When the input is collections.all.products, Shopify must instantiate every single product object in the store's database into the edge node's RAM. For a store with 5,000 products, this means allocating 5,000 complex objects, then performing approximately $5,000 \times \log_2(5,000) \approx 61,000$ comparison operations in a single-threaded Liquid execution environment.The impact on Time to First Byte (TTFB) is significant. While a standard Shopify page might render in 200–400ms, a page performing a | sort on a large catalog can exceed several seconds, leading to gateway timeouts (504 errors) or high bounce rates. Liquid's | sort is not stable and lacks access to database-level indices. It sorts based on the serialized value of the property, which is exponentially slower than the SQL-level sorting performed by Shopify's core engine when using collection handles or native search parameters.Shopify limits collections.all.products to the first 1,000 products by default, but even sorting 1,000 items in Liquid is roughly 100x slower than loading a pre-sorted collection. For any use case requiring a specific order (Price, Newest, etc.), the sorting must be offloaded to the database.Detection Logic (AST)Identify LiquidFilter nodes with name === "sort" or name === "sort_natural". The AST walker flags every instance of these filters in LiquidVariable chains.Trace the input source. Traverse the AST backward from the filter node to find the origin of the array:CRITICAL: The source is collections.all or collections.all.products.HIGH: The source is collection.products without a preceding {% paginate %} boundary.MEDIUM: The source is a variable derived from a | where filter on a large collection.Check for "Double Materialization". If | sort is preceded by | where, the performance penalty is compounded: first the catalog is loaded to filter it, then the filtered subset is cloned to be sorted.Verify Location. If this occurs in theme.liquid or a global section, the severity is escalated as it impacts the entire site's TTFB.Why regex fails here: A regex cannot distinguish between sorting a small static array (e.g., {% assign colors = "red,blue,green" | split: ',' | sort %}) and sorting the entire product catalog. The AST walker identifies the type and source of the object being sorted, allowing it to ignore harmless sorts while flagging critical performance bottlenecks.❌ Anti-PatternExtrait de code{%- comment -%}
ANTI-PATTERN: Sorting the entire catalog in Liquid.

Failure mode 1: Instantiates the full catalog (O(n)).
Failure mode 2: Sorts 1,000+ items in memory (O(n log n)).
Failure mode 3: Drastic increase in TTFB / risk of 504 Timeouts.
{%- endcomment -%}

{%- assign sorted_products = collections.all.products | sort: 'price' -%}

<div class="deal-finder">
  <h2>Our Best Prices</h2>
  <div class="grid">
    {%- for product in sorted_products limit: 4 -%}
      {% render 'product-card', product: product %}
    {%- endfor -%}
  </div>
</div>
✅ Optimized FixExtrait de code{%- comment -%}
  FIX: Use a pre-sorted collection curated in the Admin.
  
  Change 1: Access a specific collection (e.g., 'all-products-price-asc').
  Shopify's database maintains an index for this sort order, making 
  retrieval O(1) relative to the sort operation.
  
  Change 2: Use the native `?sort_by=` parameter for dynamic sorting,
  which filters at the database level before Liquid starts.
{%- endcomment -%}

{%- comment -%}
'all-products-by-price' is a collection with "Sort: Price: Low to High"
enabled in the Shopify Admin.
{%- endcomment -%}
{%- assign deals_collection = collections['all-products-by-price'] -%}

<div class="deal-finder">
  <h2>Our Best Prices</h2>
  <div class="grid">
    {%- if deals_collection != blank -%}
      {%- for product in deals_collection.products limit: 4 -%}
        {%- render 'product-card', 
          product: product,
          lazy_load: true 
        -%}
      {%- endfor -%}
    {%- endif -%}
  </div>
</div>
ReferencesShopify — sort filter documentationShopify — Collection sorting and filteringShopify — Performance: Avoid collections.all.productsLiquid — Performance bottlenecks and memory limits
