# Integration Blueprint - OrderCloud and Send

[OrderCloud](https://ordercloud.io/) is an API-first commerce platform and [Sitecore Send](https://moosend.com/) (previously Moosend) is a email campaign automation platform. 

The Ordercloud API powers product catalogs, user heirarchies, and order transactions behind the scenes while shoppers interact with a totally custom-build storefront front end experience. As they login, view products, add to cart, and purchase, shoppers generate valuable data about their interests. This data can provide intelligence for personalized email messaging through Sitecore Send. This enable key commerence marketing use cases like
- Timely and specific reminder emails for abandoned carts
- Segementing users by interest for marketing campaigns
- Embedding product advertisements uniquely recommended for the recipient

The key to implementing these is to add Sitecore Send's [JS tracking library](https://help.moosend.com/hc/en-us/articles/115002454009-How-can-I-install-website-tracking-by-using-the-JS-tracking-library-) to your FE storefront powered by Ordercloud. 


## Identify 

```ts
import { Auth, Me } from "ordercloud-javascript-sdk";
 
// OrderCloud Login 
var authResp = await Auth.Login("<username>", "<password">, "<clientID>", []);
// OrderCloud get current user details
var user = await Me.Get(authResp.access_token);
// Identify the current user to Sitecore Send for all following events. Sets a cookie. 
mootrack("identify", user.Email); // Also call this anywhere else a user identifies themselves with an email (register, checkout).
```

## View Product 

```ts
import { Me } from "ordercloud-javascript-sdk";

// OrderCloud product detail page
var product = await Me.GetProductAsync("<productID>");
// Forward the event to Sitecore Send
mootrack("PAGE_VIEWED", [{
    itemCode: product.ID,
    itemName: product.Name,
    itemImage: product.xp.Images[0].Url,
    itemUrl: product.xp.Url
}];
```

## Add To Cart

```ts
import { LineItems } from "ordercloud-javascript-sdk";

// OrderCloud Add to cart
var lineItem = await LineItems.Create("Outgoing", "<orderID>", {
    ProductID: "<unique-product-code>",
    Quantity: 3
});
// Forward the event to Sitecore Send
mootrack(
    "trackAddToOrder", 
    lineItem.Product.ID, 
    lineItem.UnitPrice, 
    lineItem.Product.xp.url, 
    lineItem.Quantity
)

```

## Purchase 

```ts
import { Orders, LineItems } from "ordercloud-javascript-sdk"

// Order LineItems saved in browser memory
var lineItems: LineItem[];
// OrderCloud submit order
await Orders.Submit("Outgoing", "<orderID>")
var products = lineItems.map(lineItem => 
    // Convert product model from OrderCloud to Sitecore Send. See Appendix.
    Convert(lineItem));
// Forward a list of ordered products to Sitecore Send
mootrack("trackOrderCompleted", products);

```

## Converting Product Models 

```ts
import { LineItem } from "ordercloud-javascript-sdk"

interface SitecoreSendProduct {
    itemCode?: string;
    itemName?: string;
    itemImage?: string; 
    itemPrice?: number;
    itemUrl?: string; 
    itemQuantity?: number;
    itemTotalPrice?: number;
    itemCategory?: string; 
    itemManufacturer?: string;
    itemSupplier?: string;
    myProperty?: any;
}

// Convert an OrderCloud lineItem to SiteCore Send's product model
function Convert(lineItem: LineItem): SitecoreSendProduct {
    return {
        itemCode: lineItem.Product.ID,
        itemName: lineItem.Product.Name,        
        itemImage: lineItem.Product.xp.Images[0].url, // depends on defining this extended property (xp)
        itemPrice: lineItem.UnitPrice,
        itemUrl: lineItem.Product.xp.url,  // depends on defining this extended property (xp)
        itemQuantity: lineItem.Quantity,
        itemTotalPrice: lineItem.LineTotal,       
        itemCategory: lineItem.Product.xp.category,  // depends on defining this extended property (xp)        
        itemManufacturer: lineItem.Product.xp.manufacturer, // depends on defining this extended property (xp)
        itemSupplier: lineItem.Product.xp.supplier, // depends on defining this extended property (xp)        
        myProperty: lineItem.Product.xp.myProperty // any custom property for segmentations or automations
    }
}
```