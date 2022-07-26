## Client Side cart module

[[toc]]

Larammerce base theme has a build-in JS module named `LocalCartService` to manage the client-side cart in the customer's browser.
This document reviews the life cycle and progress of this module.


`LocalCartService` is placed in the `resources/assets/js/define/local_cart_service.js` file consisting of 437 lines of code. It is based on require js module. To read more about it, refer to this address:[requirejs](https://www.requirejs.org/).

First, let's check how `LocalCartService` manages the shopping cart:

The image below is one of the websites created with Larammerce, and it can be seen that if the user is not logged in, the module manages the shopping cart, and if the user selects a product, it is added to the shopping cart.

As you should know, when the user is not logged in, the shopping cart data is stored as JSON in the cookie, and if the user is logged in, the cart data will be saved simultaneously on the client-side (cookie) and server-side (database).

![LocalCartService.png](/LocalCartService.png)

To see how the products are stored in the shopping cart on the client-side, copy the value of the local_cart_vn (as n is the version of the client cookie management), and decode it as it's encoded by default by the browser in url_encode format. As a result, the following value should be resulted (A JSON object with product ids as the key and value filled by the count of each product.):

```json
{
	"673": {
		"count": 10
	},
	"676": {
		"count": 1
	}
}
```
There are two products in the shopping cart with the keys 673 and 676, which are the product IDs, and the count value is the number of products stored in the shopping cart.

After logging in, server side requests are checked to see how the data is stored on the server:

![LocalCartService-update-count.png](/LocalCartService-update-count.png)

After increasing the number of products, a request is sent to the server. In the request address, it can be seen that the number of products in the cart has been updated, and the number of products has increased to 11.



Now which ajax request is called in the shopping cart process is checked:

![LocalCartService-shopping-cart-process.png](/LocalCartService-shopping-cart-process.png)

In the above image, it can be seen that there were ten products in the shopping cart at first. After reducing the number of products, This request: `/customer/cart/update-count/673?count=7` is sent to the server, which means that the number of products in the shopping cart will be updated.

If the request to decrease or increase the number of products is made consecutively, only the last request will be sent to the server.

When the product is removed from the shopping cart, the following request is sent to the server:
`/customer/cart/detach-product/673`
And after adding to the shopping cart, the following request will be sent:
`/customer/cart/attach-product/673`

If the product is deleted from the shopping cart, the Detach ajax request is called, and after adding a product to the shopping cart, the Attach ajax request is called.

As mentioned, when the user is logged in, three ajax requests will be called, and the data to be saved on the server-side (aka database).:

- add to cart
- remove from call
- update product count.

And if the user logs out, the shopping cart data will be deleted from the client side (the cookie). So the cookie value for local_cart_vn will be empty.
As you know, when the user is logged out, if a product is added to the shopping cart, a request will not be sent to the server and will only be stored in the cookie.


Let's check the `LocalCartService` module:

requirejs has two very important functions called `define` and `require`.

As for the define function, which is used in the first line of this file, it takes three parameters as input parameters. The first input is the name of the module; The second input is an ArrayList of the dependencies of this module and the third input is the body function of the module.

```js
define('local_cart_service', ['jquery', 'jq_cookie', 'tools', 'template', 'underscore'],
    function (jQuery, cookie, tools, template, _) {
```

In the next part, there are constants of the module.

`cartCountEl` constant shows the number of products in the cart with the cart-count selector.
```js
 const cartCountEl = jQuery('.cart-count');
```

In the box where the product content is presented, and the operations of adding to the shopping cart and removing from the shopping cart are performed, the `product-box` attribute must be given to its HTML tag.
```js
   const productSelector = '[product-box]';
```

The window.siteEnv object is provided by the backend system consisting of the environment variables placed in the .env file, starting with `SITE_`, managed by the system administrator so that the front-end programmer can have access to this kind of configuration to create a more dynamic code structure.
```js
const cartCookie = window.siteEnv.SITE_LOCAL_CART_COOKIE_NAME;
```

As mentioned in the rfc2965, the cookie storage has a limit of 4096KB, So the programmer must set a limit for the count of cart rows stored in the cookie storage.
This site environment value helps the front-end programmer to get this limitation amount from the backend system.
```js
const cartCountLimit = window.siteEnv.SITE_LOCAL_CART_COUNT_LIMIT;
```

As for every row of the cart, there is a calculation of discount in which each row calculates the amount by itself. Still, in case of any extra discount which must be applied to the whole invoice, there should be a way to store the extra discount amount somewhere out of the contents of the row. So there is a variable named extraDiscountAmount set externally by help of the method `LocalCartService.setExtraDiscount(amount)`.
```js
let extraDiscountAmount = 0;
```

In every invoice, it is evident that there would be some extra fees consisting of shipment fees or something like that. So to keep an eye on that and have the proper calculations on the invoices, it's necessary to have the `extraFeeAmount` variable .
This variable is filled with the value named window.extraFee, provided by the backend at first. In case of any demand to change, it is modified by the method `LocalCartService.setExtraFee(amount)`.
```js 
let extraFeeAmount = window.hasOwnProperty("extraFee") ? window.extraFee : 0;
```

Now, in this section, the methods of this module are checked.

### setExtraFeeAmount
Sometimes, another module requests to set another extra fee on the invoice. For example, the shipping cost of a certain city is higher, so the `setExtraFeeAmount` function takes a new amount from the input and makes the `ExtraFeeAmount` variable equal to that new amount, and finally, the new invoice calculates.

```js
setExtraFeeAmount: function (amount) {
        extraFeeAmount = amount;
        LocalCartService.calculateInvoice();
    }
```

### setExtraDiscountAmount
This function is used when a discount is applied to the total shopping cart.

```js
setExtraDiscountAmount: function (amount) {
        extraDiscountAmount = amount;
        LocalCartService.calculateInvoice();
    },
```

### updateSumProductPrice

This function calculates the total price of the product. For example, the user adds 2 of the same product to the shopping cart with `ID: 740`, `sumPriceBefore: 200`, `sumPriceRow`: 160. And if there is a tax, the tax is also calculated and finally replaces the amount of the product. When the values before and after the discount are equal, it does not show the value before the discount. Otherwise, it shows.

`formatPrice` formats the price, so it first keeps only the number, removes the rest, converts the number to Farsi, and finally adds a comma after three digits.

```js
updateSumProductPrice: function (id, sumPriceRow, sumPriceBefore = 0) {
        const sumPriceContainer = jQuery('[data-product-id=' + id + '] .sum-price .price-data');
        const sumPriceBeforeContainer = jQuery('[data-product-id=' + id + '] .sum-price-before .price-data');
        const taxPriceContainer = jQuery('[data-product-id=' + id + '] .tax-price .price-data');

        if (sumPriceContainer.length > 0) {
                    sumPriceContainer.text(sumPriceRow);
                    sumPriceContainer.formatPrice();

                    taxPriceContainer.text(parseInt(sumPriceRow * 0.00));
                    sumPriceContainer.formatPrice();
                }
        if (sumPriceBeforeContainer.length > 0) {
                    sumPriceBeforeContainer.text(sumPriceBefore);
                    sumPriceBeforeContainer.formatPrice();
                }

        if (sumPriceRow === sumPriceBefore) {
                    sumPriceBeforeContainer.fadeOut();
            } else {
                    sumPriceBeforeContainer.fadeIn();
                }
        },
```



###  directCalculateInvoice

In this function, foreach is performed on elements that have `product-box` attributes. First, it takes the value of `data-product-id`, finds `discountContainerEl`, `priceWithoutDiscount`, and `count`. After receiving `count`, it converts it to English.

Finds the `data-discountGroup`, which is JSON data. For example, this JSON data has value = 10 and is_percetage = 1, meaning a 10% discount is given for this product, and expiration_date, meaning valid until a specific time and includes various other items.

Then it specifies `SpecialDiscount`, a special discount for a product. If the product has a `specialdiscount`, it considers this amount. Otherwise, it performs `discountGroup`.

Sometimes, the discount percentage is changed for a higher purchase amount; remove the currently active class with the `removeClass` function and add the active class with the `addClass` function. But only the final discount percentage is displayed on the cart page.

```js
directCalculateInvoice: function () {
    let finalPriceBeforeDiscount = 0;
    let finalPriceAfterDiscount = 0;
    let loadedIds = [];

    jQuery(productSelector).each(function () {
        const thisEl = jQuery(this);
    const pId = thisEl.data('product-id');
    const discountContainerEl = thisEl.find(".discount-container");
    let priceWithoutDiscount = parseInt(thisEl.attr('product-price'));
    let count = jQuery('[data-product-id=' + pId + '] .counter-box-' + pId + ' .count-control').val();
    count = tools.convertNumberToEnglish(count);

     const discountGroup = thisEl.data("discount-group");
    let isDiscountPercentage = false;
    let discount = 0;
    const specialDiscount = discountContainerEl.find(".price-data.discount-value.special");
    if (specialDiscount.length > 0) {
        discount = specialDiscount.data("amount");
    } else if (discountGroup !== null && typeof discountGroup !== "undefined") {
         isDiscountPercentage = discountGroup.is_percentage;
        discount = tools.calculateDiscount(discountGroup, count, priceWithoutDiscount);
    if (discountContainerEl.hasClass("discount-list")) {
                            discountContainerEl.find("li.active").removeClass("active");
        discountContainerEl.find(`li[data-discount-value='${discount}']`).addClass("active");
    } else {
        discountContainerEl.find(".discount-value").text(tools.convertNumberToPersian(`${discount}`));
     }
}

```
The next step calculates `SumProductPrice` for each product and updates the value. If `count>0`, it calculates the discount according to the number of products, and the value is updated. Otherwise, the discount is applied for one product.
 
```js

   loadedIds.push(`${pId}`);
         if (count > 0) {
            let priceAfterDiscount = count * (priceWithoutDiscount - (isDiscountPercentage(priceWithoutDiscount * discount / 100) : discount));
            priceWithoutDiscount = count * priceWithoutDiscount;
            finalPriceBeforeDiscount += priceWithoutDiscount;
            finalPriceAfterDiscount += priceAfterDiscount;
            LocalCartService.updateSumProductPrice(pId, priceAfterDiscount, priceWithoutDiscount);
        } else {
            let priceAfterDiscount = priceWithoutDiscount - (isDiscountPercentage(priceWithoutDiscount * discount / 100) : discount);
            LocalCartService.updateSumProductPrice(pId, priceAfterDiscount, priceWithoutDiscount);
         }

     });
```

After calculating the final price before and after the discount of each product, the total cost of the products in the shopping cart is calculated and displayed at the bottom of the product page.

![directCalculateInvoice.png](/directCalculateInvoice.png)

The cartData object stores the data of the products in the shopping cart. This section returns products that are not on the product page. For example, the `cartData` keys of a shopping cart are the following array `{673,676,124}`, the products that are not on the product page are filtered, and this array `{673,676}` is returned. Finally, foreach is performed on these two IDs, and the data inside is calculated.

```js
        Object.keys(cartData).filter((iterId) => {
            return !loadedIds.includes(iterId);
        }).forEach((iterId) => {
                const iterRow = knownRows[iterId];
                if (typeof iterRow === "undefined")
                     return;
                if (iterRow.product.has_discount && iterRow.product.previous_price !== 0) {
                    finalPriceBeforeDiscount += iterRow.count * iterRow.product.previous_price;
                    finalPriceAfterDiscount += iterRow.count * iterRow.product.latest_price;
                } else {
                    const discount = iterRow.product.discount_group !== null ?
                        tools.calculateDiscount(iterRow.product.discount_group, iterRow.count, iterRow.product.latest_price) : 0;
                    finalPriceAfterDiscount += iterRow.count * (iterRow.product.latest_price -
                        ((iterRow.product.discount_group !== null && iterRow.product.discount_group.is_percentage) ? (iterRow.product.latest_price * discount / 100) : discount));
                    finalPriceBeforeDiscount += iterRow.count * iterRow.product.latest_price;
                }
             }
        );




 ```

The last part calculates the tax function, and at the end, if the total price before and after the discount is equal, it does not show the total price before the discount. Otherwise, it shows.

```js

        const beforeDiscountPriceContainer = jQuery('.invoice-sum-container .before-discount .price-data');
        const afterDiscountPriceContainer = jQuery('.invoice-sum-container .after-discount .price-data');
        const taxPriceContainer = jQuery('.invoice-sum-container .tax .price-data');
        const discountPriceContainer = jQuery('.invoice-sum-container .discount .price-data');

        if (beforeDiscountPriceContainer.length === 0 || afterDiscountPriceContainer.length === 0) return false;

        beforeDiscountPriceContainer.text(`${finalPriceBeforeDiscount}`);
        taxPriceContainer.text(`${parseInt((finalPriceAfterDiscount - extraDiscountAmount + extraFeeAmount) * 0.00)}`);
        afterDiscountPriceContainer.text(`${parseInt(finalPriceAfterDiscount - extraDiscountAmount + extraFeeAmount)}`);
        discountPriceContainer.text(`${parseInt(finalPriceBeforeDiscount - finalPriceAfterDiscount + extraDiscountAmount)}`);

        if (finalPriceAfterDiscount === finalPriceBeforeDiscount && window.currentPage !== "cart" && window.currentPage !== "invoice-payment") {
            beforeDiscountPriceContainer.fadeOut();
        } else {
            beforeDiscountPriceContainer.fadeIn();
        }

            jQuery('.price-data').formatPrice();
        },
```

### calculateInvoice

When this function is called, the time interval is 50 seconds. If no other action is taken, the `directCalculateInvoice` function is called.

```js
  calculateInvoice: function () {
                setTimeout(function () {
                    LocalCartService.directCalculateInvoice()
                }, 50);
            },
```


### updateCartCountBadge 

It takes the data in the shopping cart, converts it numerically, and then updates it.

```js
 updateCartCountBadge: function () {
                cartCountEl.removeClass("hidden");
                cartCountEl.html(Object.keys(cartData).length);
                cartCountEl.numericalData();
            },

```
### updateCartCount
This function changes the desired product number. First, it finds the product number with the desired ID in the cookie, replaces it with the new number, and stores it in the cookie again. And if a `callback` request was given, the `callback` will be called at the end.

```js
 updateCartCount: function (productId, count, callback = null) {

                function localUpdate() {
                    cartData[`${productId}`] = {count: count};
                    jQuery.cookie(cartCookie, cartData, {expires: 10, path: '/'});

                    if (typeof callback === "function")
                        callback();
                }
```
As you know, when the user logs in, the request to update the product count is sent to the server, and if it is successful, `localUpdate` is called; if it fails, it shows an error in the console. And if there is no login, only `localUpdate` is called.

```js
                if (window.authUser !== null) {
                    jQuery.ajax({
                        type: 'GET',
                        url: `/customer/cart/update-count/${productId}`,
                        data: {
                            count: count
                        }
                    }).done(function (_result) {
                        localUpdate();
                    }).fail(function (_error) {
                        console.log(_error);
                    });
                } else {
                    localUpdate();
                }
            },
```


### getRow

The function takes the `productId` from the input and returns the data of the same table row.

```js
  getRow: function (productId) {
                return cartData[`${productId}`];
            },
```

### isInCart

Checks whether the desired product is in the shopping cart or not.
```js
  isInCart: function (productId) {
                return `${productId}` in cartData;
            },
```

### delFromCart

This function removes the desired product from the shopping cart. After confirming the deletion of the product, if the user is logged in, an Ajax request is sent to the server, and then the `localdel` function is called. If the user is not logged in, the `localdel` function is called without a request, and the shopping cart data is deleted from the cookie.

```js
delFromCart: function (productId, accept = null, deny = null) {
                if (!(productId in cartData))
                    return false;

                function localDel() {
                    delete cartData[`${productId}`];
                    jQuery.cookie(cartCookie, cartData, {expires: 10, path: '/'});

                    LocalCartService.updateCartCountBadge();

                    if (typeof accept === "function")
                        accept();

                    if (window.currentPage === "cart") jQuery('[data-product-id=' + productId + ']').remove();
                }

                window.customConfirm("آیا از پاک کردن این محصول از سبد خرید خود اطمینان دارید ؟", function () {
                    if (window.authUser !== null) {
                        jQuery.ajax({
                            type: "GET", url: `/customer/cart/detach-product/${productId}`,
                        }).done(function (_result) {
                            localDel();
                        }).fail(function (_result) {
                            console.error(_result);
                        });
                    } else {
                        localDel();
                    }
                }, function () {
                    if (typeof deny === "function")
                        deny();
                });
            },
```


### addToCart

This function is for adding products to the shopping cart. The functions of `localAdd` and `showModal` are located in this function.

The `localAdd` function calls `updateCartCountBadge` and is the `showModal` function for pop-ups on the website.

```js
addToCart: function (productId, callback = null) {
                cartData[`${productId}`] = {count: 1};
                let errorText;

                function localAdd() {
                    jQuery.cookie(cartCookie, cartData, {expires: 10, path: '/'});
                    LocalCartService.updateCartCountBadge();

                    if (typeof callback === "function")
                        callback();
                }

                function showModal(message) {
                    let modalEl = jQuery("#added-to-cart-modal");
                    modalEl.find('p.question').html(message);
                    modalEl.modal('show');
                }
```
This function considers the limit for adding to the shopping cart. It calls the number from the `cartCountLimit` function. If it is greater than the number, it shows an error. Otherwise, it adds the product to the shopping cart. If the user is logged in, the Ajax request will be sent to the server, and the local function will be called. And if the user is not logged in, the function will be called and stored in the cookie.

```js
                if (Object.keys(cartData).length <= cartCountLimit) {
                    if (window.authUser === null) {
                        localAdd();
                    } else {
                        jQuery.ajax({
                            type: "GET", url: `/customer/cart/attach-product/${productId}`,
                        }).done(function (_result) {
                            localAdd();
                        }).fail(function (_result) {
                            console.error(_result);
                            if (_result.responseJSON.transmission.messages[0]) {
                                showModal("این محصول قبلا به سبد خرید شما اضافه شده است.");
                            }
                        });
                    }
                } else {
                    errorText = template.localCartTypeLimitError({count_limit_basket: cartCountLimit});
                    showModal(errorText);
                }
            },

```


### initProductElement
This function is the most important function of shopping cart management. It includes functions such as showing and hiding the button, fixMaxVal, update, Or what happens after clicking the increase or decrease buttons.
For example, the update function in this function calls `addToCart`. Of course, if the count is not more than the allowed number of purchases.

```js
  initProductElement: function (productElement) {
                const productId = parseInt(productElement.data('product-id'));
                const addToCartButton = productElement.find(".add-basket");
                const counterBox = productElement.find('div[class^="counter-box-"]')
                const countInput = productElement.find('.count-group input.count-control-local');
                const increaseButton = productElement.find('.count-group .count-increase');
                const decreaseButton = productElement.find('.count-group .count-decrease');
                countInput.minVal = countInput.attr('data-min') ? parseInt(countInput.data('min')) : 1;
                countInput.maxVal = countInput.attr('data-max') ? parseInt(countInput.data('max')) : 2;

                function showAddButton() {
                    if (addToCartButton.length > 0) {
                        counterBox.addClass("d-none");
                        addToCartButton.removeClass("d-none");
                    }
                }

                function hideAddButton() {
                    if (addToCartButton.length > 0) {
                        counterBox.removeClass("d-none");
                        addToCartButton.addClass("d-none");
                    }
                }

                increaseButton.on('click', function (event) {
                    event.preventDefault();
                    let thisCount = countInput.val();
                    thisCount = tools.dropNonDigits(thisCount);
                    thisCount = tools.convertNumberToEnglish(thisCount);
                    thisCount = parseInt(thisCount);
                    thisCount += 1;
                    countInput.val(thisCount.toString());
                    countInput.trigger('change');

                    return false;
                });

                decreaseButton.on('click', function (event) {
                    event.preventDefault();
                    let thisCount = countInput.val();
                    thisCount = tools.dropNonDigits(thisCount);
                    thisCount = tools.convertNumberToEnglish(thisCount);
                    thisCount = parseInt(thisCount);
                    thisCount -= 1;
                    countInput.val(thisCount.toString());
                    countInput.trigger('change');

                    return false;
                });


                addToCartButton.on('click', function (_event) {
                    _event.stopPropagation();
                    _event.preventDefault();
                    countInput.val("1");
                    countInput.trigger('change');
                    hideAddButton();
                    return false;
                });

                productElement.find("div.delete > a.del-product").on('click', function (_event) {
                    _event.stopPropagation();
                    _event.preventDefault();
                    LocalCartService.delFromCart(productId, function () {
                        showAddButton();
                        LocalCartService.calculateInvoice();
                    });
                    return false;
                });

                countInput.on('change keyup', function (event) {
                    event.preventDefault();

                    window.cartRowCountUpdateTimeouts = window.cartRowCountUpdateTimeouts || {};
                    clearTimeout(window.cartRowCountUpdateTimeouts[productId]);

                    let thisCount = countInput.val();
                    thisCount = tools.dropNonDigits(thisCount);
                    countInput.val(tools.convertNumberToPersian(`${thisCount}`));
                    thisCount = parseInt(tools.convertNumberToEnglish(thisCount));

                    if (`${productId}` in cartData && cartData[`${productId}`].count === thisCount)
                        return;

                    function fixMaxVal() {
                        if (thisCount > countInput.maxVal) {
                            window.customAlert("حد اکثر تعداد مجاز خرید این محصول " + tools.convertNumberToPersian(countInput.maxVal.toString()) + " عدد میباشد.");
                            thisCount = countInput.maxVal;
                            countInput.val(tools.convertNumberToPersian(`${thisCount}`));
                        }
                        if (thisCount > 1)
                            LocalCartService.updateCartCount(productId, thisCount);
                    }

                    function update() {
                        if (`${productId}` in cartData) {
                            fixMaxVal();
                        } else {
                            LocalCartService.addToCart(productId, fixMaxVal);
                        }
                    }

                    window.cartRowCountUpdateTimeouts[productId] = setTimeout(function () {
                        if (thisCount < countInput.minVal || thisCount === 0) {
                            LocalCartService.delFromCart(productId, function () {
                                showAddButton();
                            }, function () {
                                thisCount = countInput.minVal === 0 ? 1 : countInput.minVal;
                                countInput.val(tools.convertNumberToPersian(`${thisCount}`));
                                update();
                            });
                        } else {
                            update();
                        }
                        LocalCartService.calculateInvoice();
                    }, 200);

                    return false;
                });


                if (productId in cartData) {
                    countInput.val(cartData[productId].count);
                    hideAddButton();
                } else {
                    showAddButton();
                }

                countInput.trigger('change');
            },
```




### init

The init function is the first function that is called and checks that it creates a `serverSideCart` if the user enters and foreaches all rows of product data. And if the processes become zero, reload the page.

```js
init: function () {
                if (window.authUser !== null) {
                    let serverSideCart = {};
                    window.userCart.forEach(function (iterRow) {
                        serverSideCart[`${iterRow.product_id}`] = {count: iterRow.count};
                        knownRows[`${iterRow.product_id}`] = iterRow;
                    });

                    let concurrentProcesses = 0;
                    let countOfNewProducts = 0;

                    function reloadPage() {
                        concurrentProcesses--;

                        if (concurrentProcesses === 0 && countOfNewProducts > 0) {
                            location.reload();
                        }
                    }
```

This part of the function synchronizes cookie data with server data. For example, if the user adds five product items to the shopping cart while not logged in, this data is stored in the cookie. After login, if two items of the same product are in the `serverSideCart`, the `updateCartCount` function is called, its count is 5, and a process is added to the cart processes.

But if the user has a product in the cookie that is not in the `serverSideCart`, the `addToCart` function is called. If the desired product count is more than one, the `updateCartCount` function is called, and finally, `serverSideCart` is merged with `cartData` and stored in the cookie.

```js
                    _.each(cartData, function (data, productId) {
                        if (productId in serverSideCart) {
                            if (data.count !== serverSideCart[productId].count) {
                                concurrentProcesses++;
                                LocalCartService.updateCartCount(productId, data.count, function () {
                                    reloadPage();
                                });
                            }
                        } else {
                            concurrentProcesses++;
                            LocalCartService.addToCart(productId, function () {
                                countOfNewProducts++;
                                if (data.count > 1) {
                                    concurrentProcesses++;
                                    LocalCartService.updateCartCount(productId, data.count, function () {
                                        reloadPage();
                                    });
                                }
                                reloadPage();
                            });
                        }
                    });

                    cartData = {...serverSideCart, ...cartData};
                    jQuery.cookie(cartCookie, cartData, {path: '/'});
                }

```
Finally, the products with the `product-box` element are checked, and all the functionalities of the `initProductElemen`t function are implemented on the products.

```js

                jQuery(productSelector).each(function () {
                    LocalCartService.initProductElement(jQuery(this));
                });

                LocalCartService.calculateInvoice();
            },
```
