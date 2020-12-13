# Overview

This document describes the process for taking a VIN as an input parameter and determining various output parameters for the use in the VIN retargeting project.

## Parameters

### Input Parametrs

- `{vin}` - string
- `{fuel_api_key}` - string
- `{frikintech_dealer_id}` - string

### Output Parameters

- `{vehicle_photo_url_sm}` - string
- `{vehicle_photo_url_lg}` - string
- `{payment_amount}` - numeric string
- `{lease_or_purchase}` - string ('lease' or 'purchase')

## External APIs

### FrikinTech

FrikinTech provides the vehicle- and payment-specific details.

**Base URL:** https://api.frikintech.com/vehicles/

#### Authentication

FrikinTech uses a custom header to authenticate requests.  With each request, include the following header:

**X-FRIKIN-ROOFTOP:** `{frikintech_dealer_id}`

#### Data Frequency

It is recommended to update data from FrikinTech daily.  Suggested time: **4am PST**.

### FUEL

FUEL provides the vehicle image URLs.

**Base URL:** https://api.fuelapi.com/v1/json/

#### Authentication

FUEL uses HTTP Basic authentication.  With every request, include the following header:

**Authorization:** Basic `{fuel_api_key}`

#### Data Frequency

It is recommended to update data from FUEL daily.  Suggested time: **4am PST**.

## Algorithm

1. Request the CSV feed text from FrikinTech:
   
   ```javascript
   var csv_string = req('https://api.frikintech.com/vehicles/export/fluency')
   ```
   

1. Parse the csv text into records:
   
   ```javascript
   var all_records = csv_parse(csv_string)
   ```

1. Find the appropriate record using the input `{vin}` parameter and the `vehicle_vin` column in the records:
   
   ```javascript
   var record = select top 1 x from all_records where vehicle_vin = {vin}
   ```

1. Capture variables from found record:

   ```javascript
   var vehicle_year = record['vehicle_year']
   var vehicle_make = record['vehicle_make']
   var vehicle_trim = record['vehicle_trim']
   var vehicle_model = record['vehicle_model']
   var vehicle_color = record['vehicle_exterior_color']
   var vehicle_condition = record['vehicle_condition']
   var loan_payment = record['loan_payment_rounded']
   var lease_payment = record['lease_payment_rounded']
   ```

1. Determine the FUEL vehicle ID by using prior captured variables:

   ```javascript
   var json_string = req('https://api.fuelapi.com/v1/json/vehicles?year={vehicle_year}&make={vehicle_make}&model={vehicle_model}&trim={vehicle_trim}')

   var parsed = parseJson(json_string)

   /* note that the above includes the vehicle trim for a specific 
   match. in many cases, the trim value provided by FrikinTech does 
   not exactly match the trim value from FUEL. if this is the case, 
   we'll get an error response. below is how you check for that */

   if (parsed.code == 'no_vehicles_found') {
       // we didn't get a match using the trim, so try without:

       json_string = req('https://api.fuelapi.com/v1/json/vehicles?year={vehicle_year}&make={vehicle_make}&model={vehicle_model}')

       parsed = parseJson(json_string)

       if (parse.code == 'no_vehicles_found') {
           /* we didn't get a match without the trim either, so 
           effectively we cannot pull an image from FUEL. this 
           shouldn't be a common occurrence, but it should be 
           monitored to see how often it happens. abort */
           return;
       }
   }

   // at this point, we found a match from FUEL; capture the {fuel_vehicle_id} variable

   var fuel_vehicle_id = parsed[0]['id']
   ```

1. Query the vehicle photos from FUEL using the captured `{fuel_vehicle_id}`

   ```javascript
   var json_string = req('https://api.fuelapi.com/v1/json/vehicle/{fuel_vehicle_id}?productID=2&color={vehicle_color}')

   var json = parseJson(json_string)

   /* we need to loop through the products to find the images with the
   correct codes. there are two codes we are interested in:

   - color_1280_032: large photo
   - color_0640_032: small photo

   */

   var vehicle_photo_url_sm;
   var vehicle_photo_url_lg;

   var products = json['products']

   // first loop through products
   for (var i = 0; i < products.length; i++) {
       var product = products[i]

       // then loop through formats
       var formats = product['productFormats']

       for (var j = 0; j < formats.length; j++) {
           var item = formats[j];

           // capture the url
           var url = item['assets'][0]['url']
           
           // check for the two different codes we want
           switch (item['code']) {
               case 'color_1280_032':
                   vehicle_photo_url_lg = url
               case 'color_0640_032':
                   vehicle_photo_url_sm = url
               default:
                   continue;
           }
       }
   }
   ```

1. Use captured variables to produce the ad variables:

   ```javascript
   var vehicle_photo_url_sm = // previously captured
   var vehicle_photo_url_lg = // previously captured

   /* these two are dependent on the vehicle_condition value
   that we previously captured. get the lowercase value of 
   vehicle_condition and check if it's 'new' or 'used' */

   var payment_amount;
   var lease_or_purchase;

   // vehicle_condition was previously captured
   switch (vehicle_condition.toLower()) {
       case 'new':
           // lease_payment was previously captured
           payment_amount = lease_payment
           lease_or_purchase = 'lease'
       default:
           // purchase_payment was previously captured
           payment_amount = purchase_payment
           lease_or_purchase = 'purchase'
   }
   ```

1. We're done!  All output parameters listed at the beginning of the document should now be assigned.