# API version  
Last updated on July, 30, 2019. Current version 0.1.  

**API contents:**  
 - [Overview](#overview)
 - [Public API](#public-api)
 - [Private API](#private-api)  

# Overview  

**Overview contents:**  
 - [Length in bytes](#length-in-bytes)
 - [Header](#header)
 - [Types](#types)
 - [Pagination](#pagination)  

Inanomo API is completely asynchronous and uses websockets. All requests and feeds are made through one interface using [MessagePack](https://msgpack.org/) serialization. The base URL is [wss://market.inanomo.com/market](wss://market.inanomo.com/market). Each API request and response consists of the following:  
 1. length in bytes
 2. header
 3. data and error (body)  

 ![alt text](img/UserAPI/1_1.png)  

## Length in bytes  
Content length, including header and data. The length is calculated from the serialized message in bytes and is set in a serialized format before the start of the request or response.  

## Header  

**Header contents:**  
 - [MessageType](#messagetype)
 - [InvocationId](#invocationid)
 - [CorrelationId](#correlationid)  

|Param|Type|Description|Mandatory|
|-|-|-|-|
|MessageType|Enum|1 — Invocation<br>3 — Completion<br>6 — Ping<br>7 — Close|Yes|
|InvocationId|String|ID to match the request with the response|Yes|
|Method|String|Called method name|Yes|
|CorrelationId|String|Service ID|No|  

### MessageType  
Enum [signalR](https://docs.microsoft.com/en-us/javascript/api/@aspnet/signalr/messagetype?view=signalr-js-latest).  

### InvocationId  
An invocation ID is an identification number that uniquely identifies a response within the stream. Each request must contain an invocation ID, and each response includes the invocation ID of its request. Regardless of the sequence in which responses come, you can determine which request the response belongs to by its invocation ID.

### CorrelationId  
Service identifier

<details >
<summary>Example</summary>  
Request for a list of currencies (GetCurrencies method)  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"0",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetCurrencies"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[]<br>]|14 92 93 01 A1 30 AD 47 65 74 43 75 72 72 65 6E 63 69 65 73 90|  

`14` — serialized request length (20 bytes)
`92 93 01 A1 30 AD 47 65 74 43 75 72 72 65 6E 63 69 65 73 90` —  GetCurrencies request  
</details>  

## Types  

**Types contents:**  
 - [Timestamps (Date Time)](#timestamps-date-time)
 - [ApiProductId](#apiproductid)
 - [Decimal](#decimal)
 - [OrderTrigger](#ordertrigger)  

### Timestamps (Date Time)  
For all datetime param use the [Unix timestamp format](https://en.wikipedia.org/wiki/Unix_time) with microseconds. The number of microseconds since the beginning of the unix era in UTC.  

<details>
<summary>Serialized example</summary>  

|Human date|JSON equivalent|Serialized|
|-|-|-|
|26 November 2018, 14:34:44.207|...<br>[<br>&nbsp;&nbsp;1543242884207576<br>]<br>...|...<br>91 CB 43 15 EE 48 EF A9 4F 60<br>...|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    /// <summary>API date and time</summary>
    [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
    public readonly struct ApiDateTime
    {
        private static readonly long EpochMicroseconds = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc).Ticks/10;

        public ApiDateTime(long unixMicroseconds)
        {
            UnixMicroseconds = unixMicroseconds;
        }

        /// <summary>The number of microseconds since the beginning of the unix era in utc</summary>
        public long UnixMicroseconds { get; }

        public static ApiDateTime Create(DateTime value)
        {
            if (value.Kind != DateTimeKind.Utc)
            {
                throw new NotSupportedException($"UTC date expected, got {value.Kind}");
            }

            var microseconds = value.Ticks / 10 - EpochMicroseconds;

            return new ApiDateTime(microseconds);
        }

        public static ApiDateTime? CreateNullable(DateTime? value) => value != null ? Create(value.Value) : (ApiDateTime?)null;

        public override string ToString() => ToDateTime().ToString("yyyy-MM-dd HH:mm:ss.ffffff");

        public DateTime ToDateTime()
        {
            return new DateTime((EpochMicroseconds + UnixMicroseconds) * 10, DateTimeKind.Utc);
        }
    }
```  
</details>  

### ApiProductId  
An array of strings. The first element of the array is the base currency, the second is the quoted currency.  

<details>
<summary>Serialized example</summary>  

|JSON equivalent|Serialized|
|-|-|
|...<br>[<br>&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;"USDT"<br>]<br>...|...<br>92 A3 42 54 43 A4 55 53 44 54<br>...|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    /// <summary>Product identifier in API</summary>
    [MessagePackObject]
    public readonly struct ApiProductId : IEquatable<ApiProductId>
    {
        public ApiProductId(
            [NotNull] string baseCurrency,
            [NotNull] string quotedCurrency
        )
        {
            BaseCurrency = (baseCurrency ?? throw new ArgumentNullException(nameof(baseCurrency))).ToUpperInvariant();
            QuotedCurrency = (quotedCurrency ?? throw new ArgumentNullException(nameof(quotedCurrency))).ToUpperInvariant();
        }

        /// <summary>Base currency</summary>
        [Key(0)]
        [NotNull]
        public string BaseCurrency { get; }

        /// <summary>Quoted currency</summary>
        [Key(1)]
        [NotNull]
        public string QuotedCurrency { get; }

        public override string ToString() => $"{BaseCurrency}/{QuotedCurrency}";

        public static ApiProductId Create(string baseCurrency, string quotedCurrency)
        {
            return new ApiProductId(baseCurrency, quotedCurrency);
        }

        public bool Equals(ApiProductId other)
        {
            return string.Equals(BaseCurrency, other.BaseCurrency, StringComparison.OrdinalIgnoreCase) &&
                   string.Equals(QuotedCurrency, other.QuotedCurrency, StringComparison.OrdinalIgnoreCase);
        }

        public override bool Equals(object obj)
        {
            if (ReferenceEquals(null, obj)) return false;
            return obj is ApiProductId other && Equals(other);
        }

        public override int GetHashCode()
        {
            unchecked
            {
                return (StringComparer.OrdinalIgnoreCase.GetHashCode(BaseCurrency) * 397) ^
                       StringComparer.OrdinalIgnoreCase.GetHashCode(QuotedCurrency);
            }
        }

        public static bool operator ==(ApiProductId left, ApiProductId right) => left.Equals(right);

        public static bool operator !=(ApiProductId left, ApiProductId right) => !left.Equals(right);

        public static ApiProductId Parse([NotNull] string s)
        {
            if (string.IsNullOrWhiteSpace(s))
            {
                throw new ArgumentException("Value cannot be null or whitespace.", nameof(s));
            }
            var arr = s.Split('/');
            if (arr.Length != 2) throw new ArgumentException("String is invalid product id");
            return new ApiProductId(arr[0], arr[1]);
        }
    }
```  
</details>  

### Decimal  
Each amount, volume and price are presented in decimal format as a string. As a separator fractional point.  

<details>
<summary>Serialized example</summary>  

|Amount|JSON equivalent|Serialized|
|-|-|-|
|1234,0000000321|...<br>"1234.0000000321"<br>...|...<br>AF 31 32 33 34 2E 30 30 30 30 30 30 30 33 32 31<br>...|
|10,52|...<br>"10.5200000000"<br>...|...<br>AD 31 30 2E 35 32 30 30 30 30 30 30 30 30<br>...|  

</details>  

### OrderTrigger  
Array with string and decimal. The first element of the array is the string in which the identifier of the parent order is transmitted. The second element of the array is decimal, in which the stop price is transferred. For example, `["L-b1ed7ae8-1186-4a34-bf5a-754d3a68174e", [3493, 5000000000] ]` for market order means that after the execution of the order L-b1ed7ae8-1186-4a34-bf5a-754d3a68174e, an market order will be created with the stop price 3493,50. After market price touches the stop price, the market order will be executed, provided that the order can be executed.  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class OrderTrigger
    {
        /// <summary>Delayed activation of the order</summary>
        /// <param name="parentId">ID of the parent order, the execution of which is expected before the creation</param>
        /// <param name="stopPrice">Stop price</param>
        [SerializationConstructor]
        public OrderTrigger(
            string parentId,
            decimal? stopPrice)
        {
            ParentId = parentId;
            StopPrice = stopPrice;
        }

        /// <summary>ID of the parent order, the execution of which is expected before the creation</summary>
        [CanBeNull]
        [Key(0)]
        public string ParentId { get; }

        /// <summary>Stop price</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal? StopPrice { get; }
    }
```  
</details>  

## Pagination  
Used by cursor pagination. Cursor pagination allows for fetching results before and after the current page of results. Methods GetOrderList and GetFills return the latest items by default. To retrieve more results subsequent requests should specify which direction to paginate based on the data previously returned. For GetFills, use the CursorToken returned in the response. For GetOrderList, use the Unix timestamp as the cursor.  

|Pagination params|Description|
|-|-|
|before|Get the records above the transferred not including the transferred.|
|after|Get the records below the transferred not including the transferred.|
|limit|Maximum number of records|  

<details >
<summary>Example</summary>  

|Usage example json|Description|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"8",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetFills"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;50<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|Get the latest 50 deals. In the response comes the cursor which can be used to request the following transactions|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"9",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetFills"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;102399981,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;50<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|Get the next 50 deals.|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"10",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetOrderList"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;50<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|Get the latest 50 orders|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"11",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetOrderList"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486174902240,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;50<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|Get 50 orders below November 29, 2018, 10:09:34.902|  
</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public readonly struct ApiCursor<TId>
        where TId: struct
    {
        [SerializationConstructor]
        public ApiCursor(
            TId? before,
            TId? after,
            int limit
        )
        {
            Before = before;
            After = after;
            Limit = limit;
        }

        [Key(0)]
        [MessagePackFormatter(typeof(CursorIdFormatter))]
        public TId? Before { get; }

        [Key(1)]
        [MessagePackFormatter(typeof(CursorIdFormatter))]
        public TId? After { get; }

        [Key(2)]
        public int Limit { get; }
    }
```  
</details>  

## Error  
If the request fails, then at the end of the response comes the text of the error.  

<details>
<summary>Failure response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"1",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d714d4-4a6e-a892-a365-f35fb3a59c45"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"An&nbsp;unexpected&nbsp;error&nbsp;occurred&nbsp;invoking&nbsp;'GetCandlesSnapshot'&nbsp;on&nbsp;the&nbsp;server."<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|7C 92 94 03 A1 36 C0 D9 24 30 38 64 37 31 34 64 34 2D 34 61 36 65 2D 61 38 39 32 2D 61 33 36 35 2D 66 33 35 66 62 33 61 35 39 63 34 35 92 C0 91 92 A0 D9 49 41 6E 20 75 6E 65 78 70 65 63 74 65 64 20 65 72 72 6F 72 20 6F 63 63 75 72 72 65 64 20 69 6E 76 6F 6B 69 6E 67 20 27 47 65 74 43 61 6E 64 6C 65 73 53 6E 61 70 73 68 6F 74 27 20 6F 6E 20 74 68 65 20 73 65 72 76 65 72 2E|  
</details>  

# Public API  

**Public API contents:**  
 - [GetCurrencies](#getcurrencies)
 - [Subscribe](#subscribe)
 - [Unsubscribe](#unsubscribe)
 - [GetLastTicks](#getlastticks)
 - [GetCandlesSnapshot](#getcandlessnapshot)
 - [GetDepth](#getdepth)
 - [GetPrice](#getprice)  

This API does not require authorization.  

## GetCurrencies  
Getting a list of all currencies market.inanomo.com. This request no contains parameters, only the header.  

![alt text](img/UserAPI/1_2.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"0",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetCurrencies"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[]<br>]|14 92 93 01 A1 30 AD 47 65 74 43 75 72 72 65 6E 63 69 65 73 90|  

</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|  

![alt text](img/UserAPI/1_3.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"1",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d71113-9a2a-b79d-bee3-156032a385ff"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USD"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|43 92 94 03 A1 31 C0 D9 24 30 38 64 37 31 31 31 33 2D 39 61 32 61 2D 62 37 39 64 2D 62 65 65 33 2D 31 35 36 30 33 32 61 33 38 35 66 66 92 91 92 92 A3 42 54 43 A3 55 53 44 92 A3 42 54 43 A4 55 53 44 54 C0|  
</details>  

<details>
<summary>C# example</summary>  

```csharp
public class ProductListResponse
    {
        /// <summary>List of currency pairs</summary>
        [Key(0)]
        public IReadOnlyCollection<ApiProductId> Products { get; set; }
    }
```  
</details>  

## Subscribe  
Subscribe to currency pair events or currency.  

### Request  
|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[Subscriptions](#subscriptions)|An array of params|Subscription type and product|Yes|  

#### Subscriptions  
|Request params|Type|Description|
|-|-|-|
|SubscriptionType|Enum|Type of events to subscribe to.<br>1 — [Ticks](#tick)<br>2 — [Depth](#depthchange)<br>3 — [OrderChange](#orderchange)<br>4 — [Fills](#fill)<br>5 — AccountSumChange|
|ApiSubscribeTo|String|Product (currency pair) or currency|  

![alt text](img/UserAPI/1_4.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"1",<br>&nbsp;&nbsp;&nbsp;&nbsp;"Subscribe"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|2A 92 93 01 A1 31 A9 53 75 62 73 63 72 69 62 65 91 91 92 92 01 92 A3 42 54 43 A4 55 53 44 54 92 02 92 A3 42 54 43 A4 55 53 44 54|  
</details>  

<details>
<summary>C# example</summary>  

```csharp
public class SubscribeRequest
    {
        /// <summary>List of subscriptions</summary>
        [Key(0)]
        public IReadOnlyCollection<SubscribeRequestItem> Subscriptions { get; set; }

        /// <summary>String</summary>
        public override string ToString() => Subscriptions != null ? string.Join("; ", Subscriptions) : string.Empty;
    }

public class SubscribeRequestItem
    {
        /// <summary>Subscription type</summary>
        [Key(0)]
        public SubscriptionType Type { get; set; }

        /// <summary>Currency pair or currency</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiSubscribeTo.Formatter))]
        public ApiSubscribeTo SubscribeTo { get; set; }

        /// <summary>Tick subscription</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem Ticks(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.Ticks, SubscribeTo = product };
        /// <summary>Order book subscription</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem Depth(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.Depth, SubscribeTo = product };
        /// <summary>Subscription to the execution of orders</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem Fills(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.Fill, SubscribeTo = product };
        /// <summary>Subscribe to change orders</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem OrderChange(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.OrderChange, SubscribeTo = product };
        /// <summary>Subscription to change the amount on the trading account</summary>
        /// <param name="currency">Currency</param>
        public static SubscribeRequestItem AccountSumChange(string currency) => new SubscribeRequestItem { Type = SubscriptionType.AccountSumChange, SubscribeTo = currency };

        /// <summary>String</summary>
        public override string ToString() => $"{Type} {SubscribeTo}";
    }

```  
</details>  

## Unsubscribe  
Unsubscribe to currency pair events or currency.  

### Request  
|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[Subscriptions](#subscriptions)|An array of params|Unsubscription type and product|Yes|  

#### Subscriptions  
|Request params|Type|Description|
|-|-|-|
|SubscriptionType|Enum|Type of events to subscribe to.<br>1 — Ticks<br>2 — Depth<br>3 — OrderChange<br>4 — Fills<br>5 — AccountSumChange|
|ApiSubscribeTo|String|Product (currency pair) or currency|  

![alt text](img/UserAPI/1_5.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"2",<br>&nbsp;&nbsp;&nbsp;&nbsp;"Unsubscribe"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|2C 92 93 01 A1 32 AB 55 6E 73 75 62 73 63 72 69 62 65 91 91 92 92 01 92 A3 42 54 43 A4 55 53 44 54 92 02 92 A3 42 54 43 A4 55 53 44 54|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
public class SubscribeRequest
    {
        /// <summary>List of subscriptions</summary>
        [Key(0)]
        public IReadOnlyCollection<SubscribeRequestItem> Subscriptions { get; set; }

        /// <summary>String</summary>
        public override string ToString() => Subscriptions != null ? string.Join("; ", Subscriptions) : string.Empty;
    }

public class SubscribeRequestItem
    {
        /// <summary>Subscription type</summary>
        [Key(0)]
        public SubscriptionType Type { get; set; }

        /// <summary>Currency pair or currency</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiSubscribeTo.Formatter))]
        public ApiSubscribeTo SubscribeTo { get; set; }

        /// <summary>Tick subscription</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem Ticks(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.Ticks, SubscribeTo = product };
        /// <summary>Order book subscription</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem Depth(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.Depth, SubscribeTo = product };
        /// <summary>Subscription to the execution of orders</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem Fills(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.Fill, SubscribeTo = product };
        /// <summary>Subscribe to change orders</summary>
        /// <param name="product">Currency pair</param>
        public static SubscribeRequestItem OrderChange(ApiProductId product) => new SubscribeRequestItem { Type = SubscriptionType.OrderChange, SubscribeTo = product };
        /// <summary>Subscription to change the amount on the trading account</summary>
        /// <param name="currency">Currency</param>
        public static SubscribeRequestItem AccountSumChange(string currency) => new SubscribeRequestItem { Type = SubscriptionType.AccountSumChange, SubscribeTo = currency };

        /// <summary>String</summary>
        public override string ToString() => $"{Type} {SubscribeTo}";
    }
```  
</details>  

## GetLastTicks  
Get current market price.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|  

![alt text](img/UserAPI/1_6.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"3",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetLastTicks"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|1E 92 93 01 A1 33 AC 47 65 74 4C 61 73 74 54 69 63 6B 73 91 91 92 A3 42 54 43 A4 55 53 44 54|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
   [PublicAPI]
    [MessagePackObject]
    public class TickListRequest
    {
        [SerializationConstructor]
        public TickListRequest(
            ApiProductId productId
        )
        {
            ProductId = productId;
        }

        [Key(0)]
        public ApiProductId ProductId { get; }

    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|[TickResponse](#tickresponse)|An array of params||
|LastNotificationId|Long|The last tick alert number|  

### TickResponse  

|Response params|Type|Description|
|-|-|-|
|TickId|Long|Tick ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Tick ​​time|
|[Price](#decimal)|Decimal|Deal price|
|[Volume](#decimal)|Decimal|Filled volume of deal|
|IsBuy|Bool|Type of the deal.<br>true — means that it is a buy<br>false — means that it is a sell|  

![alt text](img/UserAPI/1_7.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"3",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d64f95-f054-5942-ad6b-954faa185022"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5591,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543242884207576,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"6411.0400000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0140382839",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|65 92 94 03 A1 33 C0 D9 24 30 38 64 36 34 66 39 35 2D 66 30 35 34 2D 35 39 34 32 2D 61 64 36 62 2D 39 35 34 66 61 61 31 38 35 30 32 32 92 92 92 A3 42 54 43 A4 55 53 44 54 91 95 CD 15 D7 CB 43 15 EE 48 EF A9 4F 60 AF 36 34 31 31 2E 30 34 30 30 30 30 30 30 30 30 AC 30 2E 30 31 34 30 33 38 32 38 33 39 C3 C0|  

</details>  

## GetCandlesSnapshot  
Get candles for the selected period.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|[DateTime start](#timestamps-date-time)|Unix timestamp|The beginning of the period for the candle|Yes|
|[DateTime end](#timestamps-date-time)|Unix timestamp|End of candle period|No|
|Duration|String|Сandle duration.<br>1m — one minute<br>5m — five minutes<br>10m — ten minutes<br>15m — fifteen minutes<br>30m — thirty minutes<br>1h — one hour<br>4h — four hours<br>1d — one day<br>1w — one week<br>1M — one month|Yes|  

![alt text](img/UserAPI/1_8.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"4",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetCandlesSnapshot"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543219440000000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543219500000000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"1m"<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|39 92 93 01 A1 34 B2 47 65 74 43 61 6E 64 6C 65 73 53 6E 61 70 73 68 6F 74 91 94 92 A3 42 54 43 A4 55 53 44 54 CB 43 15 EE 33 1A 20 70 00 CB 43 15 EE 43 DD A9 40 00 A2 31 6D|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public class CandleRequest
    {
        [Key(0)]
        public ApiProductId ProductId { get; set; }

        [Key(1)]
        [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
        public DateTime? Start { get; set; }

        [Key(2)]
        [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
        public DateTime? End { get; set; }

        [Key(3)]
        public string Duration { get; set; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|CandleData|An array of params|-|
|[Open](#decimal)|Decimal|The first price before the start of the candle|
|LastTickId|Long|The last tick, which is taken into account when forming candles|
|LastNotificationId|Long|The last tick alert number, which is taken into account when forming candles|  

### CandleData  

|Response params|Type|Description|
|-|-|-|
|[DateTime](#timestamps-date-time)|Unix timestamp|Candle start time|
|[Low](#decimal)|Decimal|The minimum price within the candle|
|[High](#decimal)|Decimal|The maximum price within the candle|
|[Open](#decimal)|Decimal|The first price of candle|
|[Close](#decimal)|Decimal|The latest price of candle|
|[Volume](#decimal)|Decimal|The volume of trades in the candle|  

![alt text](img/UserAPI/1_9.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"4",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6538d-52d3-2d3f-975b-6f3760ac6430"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1554886800000000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5246.2448000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5259.4596000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5248.7806000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5249.9925000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"72.2705589634"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1554890400000000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5249.9925000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8237000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5249.9925000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8119000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"35.5696985489"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5248.7806000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1472536,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;656262<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|CC FA 92 94 03 A1 34 C0 D9 24 30 38 64 36 35 33 38 64 2D 35 32 64 33 2D 32 64 33 66 2D 39 37 35 62 2D 36 66 33 37 36 30 61 63 36 34 33 30 92 94 92 96 CB 43 16 18 A5 2D 85 10 00 AF 35 32 34 36 2E 32 34 34 38 30 30 30 30 30 30 AF 35 32 35 39 2E 34 35 39 36 30 30 30 30 30 30 AF 35 32 34 38 2E 37 38 30 36 30 30 30 30 30 30 AF 35 32 34 39 2E 39 39 32 35 30 30 30 30 30 30 AD 37 32 2E 32 37 30 35 35 38 39 36 33 34 96 CB 43 16 18 A8 87 D3 A0 00 AF 35 32 34 39 2E 39 39 32 35 30 30 30 30 30 30 AF 35 32 35 30 2E 38 32 33 37 30 30 30 30 30 30 AF 35 32 34 39 2E 39 39 32 35 30 30 30 30 30 30 AF 35 32 35 30 2E 38 31 31 39 30 30 30 30 30 30 AD 33 35 2E 35 36 39 36 39 38 35 34 38 39 AF 35 32 34 38 2E 37 38 30 36 30 30 30 30 30 30 CE 00 16 78 18 CE 00 0A 03 86 C0|  

</details>  

## GetDepth  
Get order book  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|  

![alt text](img/UserAPI/1_10.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"5",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetDepth"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|1A 92 93 01 A1 35 A8 47 65 74 44 65 70 74 68 91 91 92 A3 42 54 43 A4 55 53 44 54|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    /// <summary>Request for a depth</summary>
    [PublicAPI]
    [MessagePackObject]
    public class DepthSnapshotRequest
    {
        /// <summary>Currency pair</summary>
        [Key(0)]
        public ApiProductId ProductId { get; set; }
    }
```  
</details>  

### Response  
Prices bids and asks come in separate arrays.  
`[ [ApiProductId], [bid1, bid2...], [ask1, ask2...] ]`  

|Response params|Type|Description|
|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|[DepthSnapshotResponseItem](#depthsnapshotresponseitem)|An array of bids and array of asks|-|
|LastNotificationId|Long|ID of the last alert applied|
|LastMyVolumeNotificationId|Long|ID of the last user volume alert applied|  

#### DepthSnapshotResponseItem  

|Response params|Type|Description|
|-|-|-|
|[MarketVolume](#decimal)|Decimal|The total volume of orders at the price level|
|[Price](#decimal)|Decimal|-|
|[MyVolume](#decimal)|Decimal|The total volume of orders of an authorized user at the price level|  

![alt text](img/UserAPI/1_11.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"5",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6be51-745b-aeb4-8fc9-f24993f26f45"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USD"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0398483146",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8076000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000000000"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0132827186",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8094000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000000000"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0265651040",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8114000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000000000"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0398482067",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8127000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000000000"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7593934,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|CC EC 92 94 03 A1 35 C0 D9 24 30 38 64 36 62 65 35 31 2D 37 34 35 62 2D 61 65 62 34 2D 38 66 63 39 2D 66 32 34 39 39 33 66 32 36 66 34 35 92 95 92 A3 42 54 43 A3 55 53 44 92 93 AC 30 2E 30 33 39 38 34 38 33 31 34 36 AF 35 32 35 30 2E 38 30 37 36 30 30 30 30 30 30 AC 30 2E 30 30 30 30 30 30 30 30 30 30 93 AC 30 2E 30 31 33 32 38 32 37 31 38 36 AF 35 32 35 30 2E 38 30 39 34 30 30 30 30 30 30 AC 30 2E 30 30 30 30 30 30 30 30 30 30 92 93 AC 30 2E 30 32 36 35 36 35 31 30 34 30 AF 35 32 35 30 2E 38 31 31 34 30 30 30 30 30 30 AC 30 2E 30 30 30 30 30 30 30 30 30 30 93 AC 30 2E 30 33 39 38 34 38 32 30 36 37 AF 35 32 35 30 2E 38 31 32 37 30 30 30 30 30 30 AC 30 2E 30 30 30 30 30 30 30 30 30 30 CE 00 73 DF CE 00 C0|  

</details>  

## GetPrice  
Get price for currency pair 

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|[DateTime](#timestamps-date-time)|Unix timestamp|-|No|  

![alt text](img/UserAPI/1_34.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"5",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetPrice"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|1B 92 93 01 A1 35 A8 47 65 74 50 72 69 63 65 91 92 92 A3 42 54 43 A4 55 53 44 54 C0|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
 public class PriceRequest
    {
        /// <summary>Currency pair</summary>
        [Key(0)]
        public ApiProductId ProductId { get; set; }

        /// <summary>For which minute the price is requested (or null if the current price is requested)</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
        public DateTime? Minute { get; set; }
    }

```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|Price|Decimal||  

![alt text](img/UserAPI/1_35.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"5",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6be51-745b-aeb4-8fc9-f24993f26f45"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"5250.8076000000"<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|3F 92 94 03 A1 35 C0 D9 24 30 38 64 36 62 65 35 31 2D 37 34 35 62 2D 61 65 62 34 2D 38 66 63 39 2D 66 32 34 39 39 33 66 32 36 66 34 35 92 91 AF 35 32 35 30 2E 38 30 37 36 30 30 30 30 30 30 C0|  

</details>  

# Public events  

**Public event contents:**  
 - [Tick](#tick)
 - [DepthChange](#depthchange)  

## Tick  
New tick event.  

|Event params|Type|Description|
|-|-|-|
|TickId|Long|Tick ​​identifier|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|[DateTime](#timestamps-date-time)|Unix timestamp|Tick ​​time|
|[Price](#decimal)|Decimal|Deal price|
|[Volume](#decimal)|Decimal|Filled volume of deal|
|IsBuy|Bool|Type of the deal.<br>true — means that it is a buy<br>false — means that it is a sell|
|NotificationId|Long|Notification ​​identifier|  

![alt text](img/UserAPI/1_12.png)  

<details>
<summary>Event example</summary>  

|JSON event equivalent|Serialized event|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"Tick",<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6c722-02a9-b991-8a64-53cee6717b49"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2782454,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USD"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1556091119621767,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"4000.0000000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0002500000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;661434<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|6D 92 94 01 C0 A4 54 69 63 6B D9 24 30 38 64 36 63 37 32 32 2D 30 32 61 39 2D 62 39 39 31 2D 38 61 36 34 2D 35 33 63 65 65 36 37 31 37 62 34 39 91 97 CE 00 2A 74 F6 92 A3 42 54 43 A3 55 53 44 CB 43 16 1D 06 C9 B1 5A 1C AF 34 30 30 30 2E 30 30 30 30 30 30 30 30 30 30 AC 30 2E 30 30 30 32 35 30 30 30 30 30 C3 CE 00 0A 17 BA|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public class TickNotification : ITickResponse, INotification
    {
        [Key(0)]
        public long TickId { get; set; }

        [Key(1)]
        public ApiProductId ProductId { get; set; }

        [Key(2)]
        [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
        public DateTime Time { get; set; }

        [Key(3)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Price { get; set; }

        [Key(4)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Volume { get; set; }

        /// <summary>true - buy, false - sale in the market order</summary>
        [Key(5)]
        public bool IsBuy { get; set; }

        [Key(6)]
        public long NotificationId { get; set; }
    }
```  
</details>  

## DepthChange  

Order Book change event. For an authorized user, it also contains changes to his orders in the order book 

Prices and volumes bids and asks come in separate arrays.
`[ [ApiProductId], [bid1, bid2...], [ask1, ask2...] ]`  

>**Important!**  
>If a second DepthChange alert arrives with the same NotificationId, then it should be ignored.  

|Event params|Type|Description|
|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|[DepthSnapshotResponseItem](#depthsnapshotresponseitem)|An array of bids and array of asks|-|
|NotificationId|Long|Notification ​​identifier|  

### DepthSnapshotResponseItem  

|Event params|Type|Description|
|-|-|-|
|[Price](#decimal)|Decimal|-|
|[MarketVolume](#decimal)|Decimal|The total volume of orders at the price level|
|[MyVolume](#decimal)|Decimal|The total volume of orders of an authorized user at the price level. If null comes, then in myvolume there were no changes at this price level|  

![alt text](img/UserAPI/1_13.png)  

<details>
<summary>Event example</summary>  

|JSON event equivalent|Serialized event|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"DepthChange",<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d7226a-52a4-921a-8c2a-f0aca5c4ae30"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USD"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"10000.0000000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000100000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000100000"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;234<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|71 92 94 01 C0 AB 44 65 70 74 68 43 68 61 6E 67 65 D9 24 30 38 64 37 32 32 36 61 2D 35 32 61 34 2D 39 32 31 61 2D 38 63 32 61 2D 66 30 61 63 61 35 63 34 61 65 33 30 91 94 92 A3 42 54 43 A3 55 53 44 91 93 B0 31 30 30 30 30 2E 30 30 30 30 30 30 30 30 30 30 AC 30 2E 30 30 30 30 31 30 30 30 30 30 AC 30 2E 30 30 30 30 31 30 30 30 30 30 90 CC EA|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    public class DepthChangeNotification : INotification
    {
        /// <summary>Currency pair</summary>
        [Key(0)]
        public ApiProductId Product { get; set; }

        /// <summary>Buy orders</summary>
        [Key(1)]
        public IReadOnlyList<Item> Bids { get; set; }

        /// <summary>Sell orders</summary>
        [Key(2)]
        public IReadOnlyList<Item> Asks { get; set; }

        /// <summary>Notification identifier</summary>
        [Key(3)]
        public long NotificationId { get; set; }

        [MessagePackObject]
        public class Item
        {
            /// <summary>Price</summary>
            [Key(0)]
            public decimal Price { get; set; }

            /// <summary>Volume of orders at the price level</summary>
            [Key(1)]
            public decimal MarketVolume { get; set; }

            /// <summary>The total volume of orders of an authorized user at the price level. If null comes, then in myvolume there were no changes at this price level</summary>
            [Key(2)]
            public decimal? MyVolume { get; set; }
        }
    }
```  
</details>  

# Private API  

**Private API contents:**  
 - [Authentication](#authentication)
 - [GetAccountState](#getaccountstate)
 - [GetAccountStateForCurrency](#getaccountstateforcurrency)
 - [GetOrder](#getorder)
 - [GetOrderList](#getorderlist)
 - [GetFills](#getfills)
 - [CreateMarketBuy](#createmarketbuy)
 - [CreateMarketSell](#createmarketsell)
 - [CreateLimBuy](#createlimbuy)
 - [CreateLimSell](#createlimsell)
 - [CancelOrder](#cancelorder)
 - [CancelAllOrders](#cancelallorders)
 - [UpdateOrder](#updateorder)

## Authentication  
Before authentication, you need to generate an API key on the [https://market.inanomo.com/trading-accounts](https://market.inanomo.com/trading-accounts) page and copy it.  

For authorization, you need to send account identifier and API key in the WS header of the package.  

|Params|Description|
|-|-|
|AccountId|Trading account number|
|ApiKey|Trading account API key |  

## GetAccountState  
Get current trading account state. This request no contains parameters, only the header.  

![alt text](img/UserAPI/1_31.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"6",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetAccountState",<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[]<br>]|17 92 94 01 A1 36 AF 47 65 74 41 63 63 6F 75 6E 74 53 74 61 74 65 C0 90|  

</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|Currency|String|Currency name|
|[AvailableSum](#decimal)|Decimal|Available balance on the trading account in currency|
|[BlockedSum](#decimal)|Decimal|Blocked funds on a trading account in currency|
|LastNotificationId|Long|Notification ​​identifier|  

![alt text](img/UserAPI/1_15.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"6",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6c722-02e1-90e4-9a43-2eb46961b786"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"7.3567524284",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"0.0000000000",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;201490<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|52 92 94 03 A1 36 C0 D9 24 30 38 64 36 63 37 32 32 2D 30 32 65 31 2D 39 30 65 34 2D 39 61 34 33 2D 32 65 62 34 36 39 36 31 62 37 38 36 92 94 A3 42 54 43 AC 37 2E 33 35 36 37 35 32 34 32 38 34 AC 30 2E 30 30 30 30 30 30 30 30 30 30 CE 00 03 13 12 C0|  

</details>  

## GetAccountStateForCurrency  
Get a trading account balance in currency.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|Currency|String|Currency name|Yes|  

![alt text](img/UserAPI/1_14.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"6",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetAccountStateForCurrency",<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC"<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|27 92 94 01 A1 36 BA 47 65 74 41 63 63 6F 75 6E 74 53 74 61 74 65 46 6F 72 43 75 72 72 65 6E 63 79 C0 91 91 A3 42 54 43|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class TradingAccountStateRequest
    {
        [Key(0)]
        public string Currency { get; set; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|Currency|String|Currency name|
|[AvailableSum](#decimal)|Decimal|Available balance on the trading account in currency|
|[BlockedSum](#decimal)|Decimal|Blocked funds on a trading account in currency|
|LastNotificationId|Long|Notification ​​identifier|  

![alt text](img/UserAPI/1_15.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"6",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6c722-02e1-90e4-9a43-2eb46961b786"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3567524284<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;201490<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|42 92 94 03 A1 36 C0 D9 24 30 38 64 36 63 37 32 32 2D 30 32 65 31 2D 39 30 65 34 2D 39 61 34 33 2D 32 65 62 34 36 39 36 31 62 37 38 36 92 94 A3 42 54 43 92 07 CB 41 EA 94 83 37 80 00 00 92 00 00 CE 00 03 13 12 C0|  

</details>  

## GetOrder  
Get one order by id.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|Id|String|Order ​​identifier|Yes|  

![alt text](img/UserAPI/1_32.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"7",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetOrder"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-90cd0104-8404-4ee8-bcdc-75ff2a1f1480"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|39 92 93 01 A1 37 A8 47 65 74 4F 72 64 65 72 91 91 91 D9 26 4C 2D 39 30 63 64 30 31 30 34 2D 38 34 30 34 2D 34 65 65 38 2D 62 63 64 63 2D 37 35 66 66 32 61 31 66 31 34 38 30|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class OrderRequest
    {
        /// <summary>Order Id</summary>
        [Key(0)]
        public string Id { get; set; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|OrderId|String|Order ​​identifier|
|Type|Enum|Order type.<br>0 — Market<br>1 — Limit<br>2 — StopMarket<br>3 — StopLimit|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|
|ParentId|String|​​Parentt order identifier|
|IsBuy|Bool|Type of the order.<br>true — means that it is a buy<br>false — means that it is a sell|
|[StopPrice](#decimal)|Decimal|Stop order price|
|[LimitPrice](#decimal)|Decimal|Limit order price|
|[Amount](#decimal)|Decimal|-|
|[AmountFilled](#decimal)|Decimal|-|
|[FillFee](#decimal)|Decimal|Commission from the completed part of the order|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|[DateTime Created](#timestamps-date-time)|Unix timestamp|Order creation time|
|[DateTime Changed](#timestamps-date-time)|Unix timestamp|Order last modified time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled<br>null — all|
|TimeToLiveMinutes|Integer|Minutes to order cancellation|
|CancellationReason|Enum|0 — InsufficientFunds<br>1 — UserAction<br>2 — Expired<br>3 — ParentCancelled|  

![alt text](img/UserAPI/1_17.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"7",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6538d-56ce-cbf2-aed8-e21e46e99f87"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-521aae27-a615-441d-8209-8ea5208b79bc",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3820,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8800000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2154120264<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486205633368,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486205633368,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|CC 99 92 94 03 A1 37 C0 D9 24 30 38 64 36 35 33 38 64 2D 35 36 63 65 2D 63 62 66 32 2D 61 65 64 38 2D 65 32 31 65 34 36 65 39 39 66 38 37 92 91 91 DC 00 10 D9 26 4C 2D 35 32 31 61 61 65 32 37 2D 61 36 31 35 2D 34 34 31 64 2D 38 32 30 39 2D 38 65 61 35 32 30 38 62 37 39 62 63 01 90 C0 C3 C0 92 CD 0E EC CB 42 00 64 2A C0 00 00 00 92 01 CB 41 E0 0C A8 89 00 00 00 92 00 00 92 00 00 92 A3 42 54 43 A4 55 53 44 54 CB 43 15 EF 2B 8C 02 8D 60 CB 43 15 EF 2B 8C 02 8D 60 01 C0 C0 C0|  

</details>  

## GetOrderList  
Get a list of user orders.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|OrderStatus|Enum|Filter orders by status.<br>0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled<br>null — all|No|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|ApiCursor|Long|See [Pagination](#Pagination)|Yes|
|IsFinished|Bool|Advanced filter orders by status.<br>true — cancel & executed<br>false — inactive & queued<br>null — all|Yes|  

![alt text](img/UserAPI/1_16.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"7",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetOrderList"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;50<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|1B 92 93 01 A1 37 AC 47 65 74 4F 72 64 65 72 4C 69 73 74 91 94 C0 C0 93 C0 C0 32 C2|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class OrderListRequest
    {
        [Key(0)]
        public OrderStatus? Status { get; set; }

        [Key(1)]
        public ApiProductId? ProductId { get; set; }

        [Key(2)]
        public ApiCursor<DateTime> Cursor { get; set; }

        /// <summary>
        /// false = only orders in the queue and orders waiting for the trigger
        /// true = completed and canceled orders
        /// null = all orders
        /// </summary>
        [Key(3)]
        public bool? IsFinished { get; set; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|OrderId|String|Order ​​identifier|
|Type|Enum|Order type.<br>0 — Market<br>1 — Limit<br>2 — StopMarket<br>3 — StopLimit|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|
|ParentId|String|Parent order identifier|
|IsBuy|Bool|Type of the order.<br>true — means that it is a buy<br>false — means that it is a sell|
|[StopPrice](#decimal)|Decimal|Optional param. Stop order price|
|[LimitPrice](#decimal)|Decimal|Optional param. Limit order price|
|[Amount](#decimal)|Decimal|-|
|[AmountFilled](#decimal)|Decimal|-|
|[FillFee](#decimal)|Decimal|Commission from the completed part of the order|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|[DateTime Created](#timestamps-date-time)|Unix timestamp|Order creation time|
|[DateTime Changed](#timestamps-date-time)|Unix timestamp|Order last modified time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled<br>null — all|
|TimeToLiveMinutes|Integer|Minutes to order cancellation|
|CancellationReason|Enum|Optional param.<br>0 — InsufficientFunds<br>1 — UserAction<br>2 — Expired<br>3 — ParentCancelled|  

![alt text](img/UserAPI/1_17.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"7",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6538d-56ce-cbf2-aed8-e21e46e99f87"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-521aae27-a615-441d-8209-8ea5208b79bc",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3820,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8800000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2154120264<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486205633368,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486205633368,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-a4fe6a12-bae1-479d-aebf-7ba018d1ef3a",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4322,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3600000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486174902240,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543486174902240,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|CC FA 92 94 03 A1 37 C0 D9 24 30 38 64 36 35 33 38 64 2D 35 36 63 65 2D 63 62 66 32 2D 61 65 64 38 2D 65 32 31 65 34 36 65 39 39 66 38 37 92 91 92 DC 00 10 D9 26 4C 2D 35 32 31 61 61 65 32 37 2D 61 36 31 35 2D 34 34 31 64 2D 38 32 30 39 2D 38 65 61 35 32 30 38 62 37 39 62 63 01 90 C0 C3 C0 92 CD 0E EC CB 42 00 64 2A C0 00 00 00 92 01 CB 41 E0 0C A8 89 00 00 00 92 00 00 92 00 00 92 A3 42 54 43 A4 55 53 44 54 CB 43 15 EF 2B 8C 02 8D 60 CB 43 15 EF 2B 8C 02 8D 60 01 C0 C0 DC 00 10 D9 26 4C 2D 61 34 66 65 36 61 31 32 2D 62 61 65 31 2D 34 37 39 64 2D 61 65 62 66 2D 37 62 61 30 31 38 64 31 65 66 33 61 01 90 C0 C3 C0 92 CD 10 E2 CB 41 EA D2 74 80 00 00 00 92 01 00 92 00 00 92 00 00 92 A3 42 54 43 A4 55 53 44 54 CB 43 15 EF 2B 84 AE DF 80 CB 43 15 EF 2B 84 AE DF 80 01 C0 C0 C0|  

</details>  

## GetFills  
Get a list of completed user transactions.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|ApiCursor|Long|See [Pagination](#Pagination)|Yes|  

![alt text](img/UserAPI/1_18.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"8",<br>&nbsp;&nbsp;&nbsp;&nbsp;"GetFills"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;50<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|14 92 93 01 A1 38 A8 47 65 74 46 69 6C 6C 73 91 91 93 C0 C0 32|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class FillRequest
    {
        [Key(0)]
        public ApiCursor<long> Cursor { get; set; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|Tick ID|Long|Tick ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Tick time|
|OrderType|Enum|0 — Market<br>1 — Limit<br>2 — StopMarket<br>3 — StopLimit|
|IsBuy|Bool|Type of the order.<br>true — means that it is a buy<br>false — means that it is a sell|
|[Price](#decimal)|Decimal|Deal price|
|[Volume](#decimal)|Decimal|Filled volume of deal|
|[Commision](#decimal)|Decimal|Transaction fee|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|OrderId|String|Order ​​identifier|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|
|CursorToken|Long|Pagination сursor ​​identifier|  

![alt text](img/UserAPI/1_19.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"8",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d655ee-1e40-6c43-a466-a6c7ff15a261"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;27214580,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543491747516956,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4377,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6800000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3000000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3133040000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-b1ed7ae8-1186-4a34-bf5a-754d3a68174e",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;102399985<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;27214579,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543491747500458,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4377,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6800000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4000000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7510720000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-b1ed7ae8-1186-4a34-bf5a-754d3a68174e",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;102399981<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|CC FC 92 94 03 A1 38 C0 D9 24 30 38 64 36 35 35 65 65 2D 31 65 34 30 2D 36 63 34 33 2D 61 34 36 36 2D 61 36 63 37 66 66 31 35 61 32 36 31 92 91 92 9B CE 01 9F 42 F4 CB 43 15 EF 30 B5 4C 48 70 01 C2 92 CD 11 19 CB 41 F9 54 FC 40 00 00 00 92 00 CB 41 E6 5A 0B C0 00 00 00 92 01 CB 41 E7 57 CC B0 00 00 00 92 A3 42 54 43 A4 55 53 44 54 D9 26 4C 2D 62 31 65 64 37 61 65 38 2D 31 31 38 36 2D 34 61 33 34 2D 62 66 35 61 2D 37 35 34 64 33 61 36 38 31 37 34 65 90 CE 06 1A 7F F1 9B CE 01 9F 42 F3 CB 43 15 EF 30 B5 4B 46 A8 01 C2 92 CD 11 19 CB 41 F9 54 FC 40 00 00 00 92 00 CB 41 ED CD 65 00 00 00 00 92 01 CB 41 FB FA C7 E0 00 00 00 92 A3 42 54 43 A4 55 53 44 54 D9 26 4C 2D 62 31 65 64 37 61 65 38 2D 31 31 38 36 2D 34 61 33 34 2D 62 66 35 61 2D 37 35 34 64 33 61 36 38 31 37 34 65 90 CE 06 1A 7F ED C0|  

</details>  

## CreateMarketBuy  
Create a market buy order.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|[Amount](#decimal)|Decimal|-|Yes|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|No|
|OrderTrigger|Array with string and decimal|The first element of the array is the string in which the identifier of the parent order is transmitted. The second element of the array is decimal, in which the stop price is transferred|No|  

![alt text](img/UserAPI/1_20.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"9",<br>&nbsp;&nbsp;&nbsp;&nbsp;"CreateMarketBuy"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|28 92 93 01 A1 39 AF 43 72 65 61 74 65 4D 61 72 6B 65 74 42 75 79 91 94 92 A3 42 54 43 A4 55 53 44 54 92 CD 03 E8 00 90 C0|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public readonly struct CreateMarketBuyRequest : IHasTrigger
    {
        /// <summary>Parameters for creating a market order for sale</summary>
        /// <param name="productId">Currency pair id</param>
        /// <param name="amount">Transaction amount in base currency</param>
        /// <param name="tags">Tags to the order</param>
        /// <param name="trigger">Delayed activation of the order</param>
        [SerializationConstructor]
        public CreateMarketBuyRequest(
            ApiProductId productId,
            decimal amount,
            [CanBeNull] IReadOnlyList<string> tags,
            [CanBeNull] OrderTrigger trigger
        )
        {
            ProductId = productId;
            Amount = amount;
            Trigger = trigger;
            Tags = tags ?? Array.Empty<string>();
            Trigger = trigger;
        }

        /// <summary>Currency pair id</summary>
        [Key(0)]
        public ApiProductId ProductId { get; }

        /// <summary>Transaction amount in base currency</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Amount { get; }

        /// <summary>Tags to the order</summary>
        [NotNull]
        [Key(2)]
        public IReadOnlyList<string> Tags { get; }

        /// <summary>Delayed activation of the order</summary>
        [CanBeNull]
        [Key(3)]
        public OrderTrigger Trigger { get; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|ID|String|Order ​​identifier.|
|[DateTime](#timestamps-date-time)|Unix timestamp|Order creation time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled|  

![alt text](img/UserAPI/1_21.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"9",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d655ee-2395-e67a-b3cf-380f344ee32a"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"M-186d7961-d77b-4476-8311-27016636e8a5",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543822859759979,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|61 92 94 03 A1 39 C0 D9 24 30 38 64 36 35 35 65 65 2D 32 33 39 35 2D 65 36 37 61 2D 62 33 63 66 2D 33 38 30 66 33 34 34 65 65 33 32 61 92 93 D9 26 4D 2D 31 38 36 64 37 39 36 31 2D 64 37 37 62 2D 34 34 37 36 2D 38 33 31 31 2D 32 37 30 31 36 36 33 36 65 38 61 35 CB 43 15 F0 65 14 9B C5 AC 01 C0|  

</details>  

## CreateMarketSell  
Create a market sell order.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|[Amount](#decimal)|Decimal|-|Yes|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|No|
|OrderTrigger|Array with string and decimal|The first element of the array is the string in which the identifier of the parent order is transmitted. The second element of the array is decimal, in which the stop price is transferred|No|  

![alt text](img/UserAPI/1_24.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"10",<br>&nbsp;&nbsp;&nbsp;&nbsp;"CreateMarketSell"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|28 92 93 01 A2 31 30 B0 43 72 65 61 74 65 4D 61 72 6B 65 74 53 65 6C 6C 91 94 92 A3 42 54 43 A4 55 53 44 54 92 01 00 90 C0|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public readonly struct CreateMarketSellRequest : IHasTrigger
    {
        /// <summary>Parameters for creating a market order to buy</summary>
        /// <param name="productId">Currency pair id</param>
        /// <param name="amount">Transaction amount in base currency</param>
        /// <param name="tags">Tags to the order</param>
        /// <param name="trigger">Delayed activation of the order</param>
        [SerializationConstructor]
        public CreateMarketSellRequest(
            ApiProductId productId,
            decimal amount,
            [CanBeNull] IReadOnlyList<string> tags,
            [CanBeNull] OrderTrigger trigger
        )
        {
            ProductId = productId;
            Amount = amount;
            Tags = tags ?? Array.Empty<string>();
            Trigger = trigger;
        }

        /// <summary>Currency pair id</summary>
        [Key(0)]
        public ApiProductId ProductId { get; }

        /// <summary>Transaction amount in base currency</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Amount { get; }

        /// <summary>Tags to the order</summary>
        [NotNull]
        [Key(2)]
        public IReadOnlyList<string> Tags { get; }

        /// <summary>Delayed activation of the order</summary>
        [CanBeNull]
        [Key(3)]
        public OrderTrigger Trigger { get; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|ID|String|Order ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Order creation time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled|  

![alt text](img/UserAPI/1_21.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"10",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d658f6-1d63-429a-b1a5-8b95449f9b57"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"M-786178f9-f0dc-4887-812f-499bcf4e2861",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543825692879809,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|62 92 94 03 A2 31 30 C0 D9 24 30 38 64 36 35 38 66 36 2D 31 64 36 33 2D 34 32 39 61 2D 62 31 61 35 2D 38 62 39 35 34 34 39 66 39 62 35 37 92 93 D9 26 4D 2D 37 38 36 31 37 38 66 39 2D 66 30 64 63 2D 34 38 38 37 2D 38 31 32 66 2D 34 39 39 62 63 66 34 65 32 38 36 31 CB 43 15 F0 67 B8 13 AF 04 01 C0|  

</details>  

## CreateLimBuy  
Create a limit buy order.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|[Amount](#decimal)|Decimal|-|Yes|
|[Price](#decimal)|Decimal|Limit execution price|Yes|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|No|
|OrderTrigger|Array with string and decimal|The first element of the array is the string in which the identifier of the parent order is transmitted. The second element of the array is decimal, in which the stop price is transferred|No|
|cancelAfterMinutes|Integer|Minutes to order cancellation|No|  

![alt text](img/UserAPI/1_22.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"11",<br>&nbsp;&nbsp;&nbsp;&nbsp;"CreateLimBuy"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3970,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8500000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;60<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|32 92 93 01 A2 31 31 AC 43 72 65 61 74 65 4C 69 6D 42 75 79 91 96 92 A3 42 54 43 A4 55 53 44 54 92 01 00 92 CD 0F 82 CB 41 FF AA 3B 50 00 00 00 90 C0 3C|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public readonly struct CreateLimBuyRequest : IHasTrigger
    {
        /// <summary>Parameters for creating a limit order for sale</summary>
        /// <param name="productId">Currency pair id</param>
        /// <param name="amount">Transaction amount in quoted currency</param>
        /// <param name="tags">Tags to the order</param>
        /// <param name="trigger">Delayed activation of the order</param>
        /// <param name="price">Transaction price</param>
        public CreateLimBuyRequest(
            ApiProductId productId,
            decimal amount,
            decimal price,
            [CanBeNull] IReadOnlyList<string> tags,
            [CanBeNull] OrderTrigger trigger,
            int? cancelAfterMinutes)
        {
            ProductId = productId;
            Amount = amount;
            Price = price;
            Tags = tags ?? Array.Empty<string>();
            Trigger = trigger;
            CancelAfterMinutes = cancelAfterMinutes;
        }

        /// <summary>Currency pair id</summary>
        [Key(0)]
        public ApiProductId ProductId { get; }

        /// <summary>Transaction amount in quoted currency</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Amount { get; }

        /// <summary>Transaction price</summary>
        [Key(2)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Price { get; }

        /// <summary>Tags to the order</summary>
        [NotNull]
        [Key(3)]
        public IReadOnlyList<string> Tags { get; }

        /// <summary>Delayed activation of the order</summary>
        [CanBeNull]
        [Key(4)]
        public OrderTrigger Trigger { get; }

        [Key(5)]
        public int? CancelAfterMinutes { get; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|ID|String|Order ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Order creation time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled|  

![alt text](img/UserAPI/1_21.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"11",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d658f6-1d7a-11c4-9e6f-06b5cd1f30f8"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-bb64495d-05e5-4ec0-846d-9e9f22f711c1",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543829712845082,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|62 92 94 03 A2 31 31 C0 D9 24 30 38 64 36 35 38 66 36 2D 31 64 37 61 2D 31 31 63 34 2D 39 65 36 66 2D 30 36 62 35 63 64 31 66 33 30 66 38 92 93 D9 26 4C 2D 62 62 36 34 34 39 35 64 2D 30 35 65 35 2D 34 65 63 30 2D 38 34 36 64 2D 39 65 39 66 32 32 66 37 31 31 63 31 CB 43 15 F0 6B 76 82 E4 68 01 C0|  

</details>  

## CreateLimSell  
Create a limit sell order.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|Yes|
|[Amount](#decimal)|Decimal|-|Yes|
|[Price](#decimal)|Decimal|Limit execution price|Yes|
|Tags|An array of strings|May contain any text. Used for convenient grouping or labeling|No|
|OrderTrigger|Array with string and decimal|The first element of the array is the string in which the identifier of the parent order is transmitted. The second element of the array is decimal, in which the stop price is transferred|No|
|cancelAfterMinutes|Integer|Minutes to order cancellation|No|  

![alt text](img/UserAPI/1_23.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"12",<br>&nbsp;&nbsp;&nbsp;&nbsp;"CreateLimSell"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USDT"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3978,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7600000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|33 92 93 01 A2 31 32 AD 43 72 65 61 74 65 4C 69 6D 53 65 6C 6C 91 96 92 A3 42 54 43 A4 55 53 44 54 92 01 00 92 CD 0F 8A CB 41 FC 4F EC C0 00 00 00 90 C0 C0|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public readonly struct CreateLimSellRequest : IHasTrigger
    {
        /// <summary>Parameters for creating a limit order to buy</summary>
        /// <param name="productId">Currency pair id</param>
        /// <param name="amount">Transaction amount in quoted currency</param>
        /// <param name="tags">Tags to the order</param>
        /// <param name="trigger">Delayed activation of the order</param>
        /// <param name="price">Transaction price</param>
        public CreateLimSellRequest(
            ApiProductId productId,
            decimal amount,
            decimal price,
            [CanBeNull] IReadOnlyList<string> tags,
            [CanBeNull] OrderTrigger trigger,
            int? cancelAfterMinutes)
        {
            ProductId = productId;
            Amount = amount;
            Price = price;
            Tags = tags ?? Array.Empty<string>();
            Trigger = trigger;
            CancelAfterMinutes = cancelAfterMinutes;
        }

        /// <summary>Currency pair id</summary>
        [Key(0)]
        public ApiProductId ProductId { get; }

        /// <summary>Transaction amount in quoted currency</summary>
        [Key(1)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Amount { get; }

        /// <summary>Transaction price</summary>
        [Key(2)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Price { get; }

        /// <summary>Tags to the order</summary>
        [NotNull]
        [Key(3)]
        public IReadOnlyList<string> Tags { get; }

        /// <summary>Delayed activation of the order</summary>
        [CanBeNull]
        [Key(4)]
        public OrderTrigger Trigger { get; }

        [Key(5)]
        public int? CancelAfterMinutes { get; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|ID|String|Order ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Order creation time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled|  

![alt text](img/UserAPI/1_21.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"12",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d658f6-1d82-17e2-8af5-3550bf87c40c"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-90cd0104-8404-4ee8-bcdc-75ff2a1f1480",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543831123361529,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|62 92 94 03 A2 31 32 C0 D9 24 30 38 64 36 35 38 66 36 2D 31 64 38 32 2D 31 37 65 32 2D 38 61 66 35 2D 33 35 35 30 62 66 38 37 63 34 30 63 92 93 D9 26 4C 2D 39 30 63 64 30 31 30 34 2D 38 34 30 34 2D 34 65 65 38 2D 62 63 64 63 2D 37 35 66 66 32 61 31 66 31 34 38 30 CB 43 15 F0 6C C6 CD FB E4 01 C0|  

</details>  

## CancelOrder  
Cancel order by id.  

### Request  

|Request params|Type|Description|Mandatory|
|-|-|-|-|
|ID|String|Order ​​identifier|Yes|  

![alt text](img/UserAPI/1_25.png)  

<details>
<summary>Request example</summary>  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"13",<br>&nbsp;&nbsp;&nbsp;&nbsp;"CancelOrder"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-8a061860-6b05-4c05-a598-4677458ccb25"<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|3C 92 93 01 A2 31 33 AB 43 61 6E 63 65 6C 4F 72 64 65 72 91 91 D9 26 4C 2D 38 61 30 36 31 38 36 30 2D 36 62 30 35 2D 34 63 30 35 2D 61 35 39 38 2D 34 36 37 37 34 35 38 63 63 62 32 35|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [PublicAPI]
    [MessagePackObject]
    public class CancelOrderRequest
    {
        [Key(0)]
        public string Id { get; set; }
    }
```  
</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|ID|String|Order ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Order update time|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled|  

![alt text](img/UserAPI/1_21.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"13",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d658f6-1d88-45f1-9d77-c1329bdcd2dc"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-8a061860-6b05-4c05-a598-4677458ccb25",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1543832210790748,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|62 92 94 03 A2 31 33 C0 D9 24 30 38 64 36 35 38 66 36 2D 31 64 38 38 2D 34 35 66 31 2D 39 64 37 37 2D 63 31 33 32 39 62 64 63 64 32 64 63 92 93 D9 26 4C 2D 38 61 30 36 31 38 36 30 2D 36 62 30 35 2D 34 63 30 35 2D 61 35 39 38 2D 34 36 37 37 34 35 38 63 63 62 32 35 CB 43 15 F0 6D CA 11 65 70 04 C0|  

</details>  

## CancelAllOrders  
Cancel all orders. This request no contains parameters, only the header.  

![alt text](img/UserAPI/1_26.png)  

<details>
<summary>Request example</summary>  

### Request  

|Equivalent json|Serialized request|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;"14",<br>&nbsp;&nbsp;&nbsp;&nbsp;"CancelAllOrders"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[]<br>]|17 92 93 01 A2 31 34 AF 43 61 6E 63 65 6C 41 6C 6C 4F 72 64 65 72 73 90|  

</details>  

### Response  

|Response params|Type|Description|
|-|-|-|
|OrderIds|An array of strings|Orders ​​identifiers|  

![alt text](img/UserAPI/1_27.png)  

<details>
<summary>Response example</summary>  

|JSON response equivalent|Serialized response|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;"14",<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d658f6-1d98-1b9c-b254-3116bf8a9f15"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"M-7054bb22-372a-4ecc-bf92-6d4a08c9c9cd",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-1b95fb90-5563-454c-a6e3-3361f853dc9f",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"L-4dfaf36e-18cf-4185-90b8-bc39cf475537"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;null<br>&nbsp;&nbsp;]<br>]|CC A9 92 94 03 A2 31 34 C0 D9 24 30 38 64 36 35 38 66 36 2D 31 64 39 38 2D 31 62 39 63 2D 62 32 35 34 2D 33 31 31 36 62 66 38 61 39 66 31 35 92 91 93 D9 26 4D 2D 37 30 35 34 62 62 32 32 2D 33 37 32 61 2D 34 65 63 63 2D 62 66 39 32 2D 36 64 34 61 30 38 63 39 63 39 63 64 D9 26 4C 2D 31 62 39 35 66 62 39 30 2D 35 35 36 33 2D 34 35 34 63 2D 61 36 65 33 2D 33 33 36 31 66 38 35 33 64 63 39 66 D9 26 4C 2D 34 64 66 61 66 33 36 65 2D 31 38 63 66 2D 34 31 38 35 2D 39 30 62 38 2D 62 63 33 39 63 66 34 37 35 35 33 37 C0|  

</details>  

## UpdateOrder  
This method will be available soon  

# Private events  

**Private events contents:**  
 - [OrderChange](#orderchange)
 - [Fill](#fill)  

## OrderChange  
User order update event.  

|Event params|Type|Description|
|-|-|-|
|OrderId|String|Order ​​identifier|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|IsBuy|Bool|Type of the deal.<br>true — means that it is a buy<br>false — means that it is a sell|
|OrderStatus|Enum|0 — Inactive<br>1 — Queued<br>2 — PartiallyExecuted<br>3 — Executed<br>4 — Cancelled|
|[DateTime Changed](#timestamps-date-time)|Unix timestamp|Order last modified time|
|[Amount](#decimal)|Decimal|-|
|[AmountFilled](#decimal)|Decimal|-|
|[DealPrice](#decimal)|Decimal|Transaction price|
|NotificationId|Long|Notification ​​identifier|  

![alt text](img/UserAPI/1_28.png)  

<details>
<summary>Event example</summary>  

|JSON event equivalent|Serialized event|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"OrderChange",<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6c722-02aa-45ec-af73-73092bf602b3"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"M-49151e31-4eaf-47a8-9525-7bdeabc8921a",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USD"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1556103187284645,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;500000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;100000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;34853<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|CC 8A 92 94 01 C0 AB 4F 72 64 65 72 43 68 61 6E 67 65 D9 24 30 38 64 36 63 37 32 32 2D 30 32 61 61 2D 34 35 65 63 2D 61 66 37 33 2D 37 33 30 39 32 62 66 36 30 32 62 33 91 99 D9 26 4D 2D 34 39 31 35 31 65 33 31 2D 34 65 61 66 2D 34 37 61 38 2D 39 35 32 35 2D 37 62 64 65 61 62 63 38 39 32 31 61 92 A3 42 54 43 A3 55 53 44 C2 03 CB 43 16 1D 12 06 D9 0A 94 92 00 00 92 00 CE 1D CD 65 00 92 CD 03 E8 CE 05 F5 E1 00 CD 88 25|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class OrderNotification : IOrder, INotification
    {
        [Key(0)]
        public string OrderId { get; set; }

        [Key(1)]
        public ApiProductId ProductId { get; set; }

        [Key(2)]
        public bool IsBuy { get; set; }

        [Key(3)]
        public OrderStatus Status { get; set; }

        [Key(4)]
        [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
        public DateTime Changed { get; set; }

        [Key(5)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Amount { get; set; }

        [Key(6)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal AmountFilled { get; set; }

        [Key(7)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal? DealPrice { get; set; }

        [Key(8)]
        public long NotificationId { get; set; }
    }
```  
</details>  

## Fill  
Event of the appearance of a new transaction at the order of the user.  

|Event params|Type|Description|
|-|-|-|
|TickId|Long|Tick ​​identifier|
|[DateTime](#timestamps-date-time)|Unix timestamp|Tick time|
|OrderType|Enum|0 — Market<br>1 — Limit<br>2 — StopMarket<br>3 — StopLimit|
|IsBuy|Bool|Type of the deal.<br>true — means that it is a buy<br>false — means that it is a sell|
|[Price](#decimal)|Decimal|-|
|[Volume](#decimal)|Decimal|Filled volume of deal|
|[Comission](#decimal)|Decimal|Transaction fee|
|[ApiProductId](#apiproductid)|An array of strings|Сurrency pair|
|OrderId|String|Order ​​identifier|
|NotificationId|Long|Notification ​​identifier|  

![alt text](img/UserAPI/1_30.png)  

<details>
<summary>Event example</summary>  

|JSON event equivalent|Serialized event|
|-|-|
|[<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;1,<br>&nbsp;&nbsp;&nbsp;&nbsp;null,<br>&nbsp;&nbsp;&nbsp;&nbsp;"Fill",<br>&nbsp;&nbsp;&nbsp;&nbsp;"08d6c722-02aa-45f1-8061-1c2c0e9ed4d4"<br>&nbsp;&nbsp;],<br>&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2782458,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1556105480744033,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1000,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;100000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;10000000<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0,<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;0<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"BTC",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"USD"<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"M-38e98c79-4ac0-4f1d-8356-1c575ad0d257",<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8467<br>&nbsp;&nbsp;&nbsp;&nbsp;]<br>&nbsp;&nbsp;]<br>]|CC 88 92 94 01 C0 A4 46 69 6C 6C D9 24 30 38 64 36 63 37 32 32 2D 30 32 61 61 2D 34 35 66 31 2D 38 30 36 31 2D 31 63 32 63 30 65 39 65 64 34 64 34 91 9A CE 00 2A 74 FA CB 43 16 1D 14 29 A6 B1 84 00 C2 92 CD 03 E8 CE 05 F5 E1 00 92 00 CE 00 98 96 80 92 00 00 92 A3 42 54 43 A3 55 53 44 D9 26 4D 2D 33 38 65 39 38 63 37 39 2D 34 61 63 30 2D 34 66 31 64 2D 38 33 35 36 2D 31 63 35 37 35 61 64 30 64 32 35 37 CD 21 13|  

</details>  

<details>
<summary>C# example</summary>  

```csharp
    [MessagePackObject]
    public class FillNotification : INotification
    {
        [Key(0)]
        public long TickId { get; set; }

        [Key(1)]
        [MessagePackFormatter(typeof(ApiDateTimeFormatter))]
        public DateTime Time { get; set; }

        [Key(2)]
        public int OrderType { get; set; }  // TODO enum

        [Key(3)]
        public bool IsBuy { get; set; }

        [Key(4)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Price { get; set; }

        [Key(5)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Volume { get; set; }

        [Key(6)]
        [MessagePackFormatter(typeof(ApiDecimalFormatter))]
        public decimal Commission { get; set; }

        [Key(7)]
        public ApiProductId ProductId { get; set; }

        [Key(8)]
        public string OrderId { get; set; }

        [Key(9)]
        public long NotificationId { get; set; }
    }
```  
</details>  