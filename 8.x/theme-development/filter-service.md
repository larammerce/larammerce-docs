## Filter Service

[[toc]]

This document reviews the life cycle and progress of the `FilterService` module. This module has build-in JS to Loading and filter products.

`FilterService` is placed in the `resources/assets/js/define/filter-service.js` file consisting of 451 lines of code.

First, let's check what section the `FilterService` module manages:

The image below is one of the websites created with Larammerce. This module manages six parts of this page: 

1- Filters

2- Sorting

3- product Count Container

4- filter Tags Container

5- products result Container

6- Url

![FilterService.png](/FilterService.png)

As seen in the picture, the **can be used in the restaurant** filter is selected, so all the products related to this filter are displayed, the **restaurant** tag is placed in the active filters section, the total number of products is counted and the URL of the selection  filter is added.

If this link is given to someone else, after entering it, this filter will be selected, and the selected products will be displayed. This feature is because the `FilterService`  module uses the `hashchange` module.To read more about `hashchange` module, refer to this address:[hashchange](http://benalman.com/code/projects/jquery-hashchange/docs/).

Now, let's check the `FilterService` module:

This module is defined as `requirejs` and has dependencies that can be seen in the code below.

```js
define('filter_service',
    ['jquery', 'underscore', 'template', 'settings', 'local_cart_service', 'tools', 'init_settings', 'hashchange', 'price_data', 'rating_service',
    ],
```

In the next part, private variables are defined.


### Private variables

`CurrentDirectoryId` variable shows what category of products you are in.
```js
var currentDirectoryId = null;   
```

`ProductCountContainer` variable is the number of products shown in the result container.
```js
var productCountContainer = 0;
```

`FilterTagsContainer` variable is for the tags displayed in the filter tag container.
```js
var filterTagsContainer = null;
```

`LoadingContainer` variables are the elements that are displayed when the product container is scrolled. They don't have any products, and they do the loading process.
```js
var loadingContainer = null;  
```

`resultContainer` variables are the filtered products that will be displayed.
```js
var resultContainer = null; 
```

When a filter is selected, and the system is loading, clicking on another filter should not be possible, and a protected layer will be activated.
```js
var loadingProtectorLayer = null; 
```

`LoadResult` variable is an array that stores the Ajax output.
```js
var loadedResult = null; 
```

`fetchLock` variable is a boolean that indicates whether the products are being loaded or not.
```js
var fetchLock = false;
```

This variable saves the selected value.
```js
var selectedValues = [];
```

This variable is for selected colors.
```js
var colors = []; 
```

This variable checks if the module is initialized or not. Initialization means first checking the URL's filters, what kind of sorting it has to dictate its containers, or whether there is an active filter.
```js
var isInitiated = false; #Checks if the module 
```

This variable will be removed in the next version.
```js
var discount = false; 
```
After selecting a filter, if there are no products to display, an image shows no results found. In this part of the SVG file, the same image is placed as text, which is wrong and should be placed in an underscore file and used only here.To read more about `underscore`, refer to this address:[underscore](http://underscorejs.org/themplate).
```js
var noResult =<div> ... </div>; 
```

There are two types of filters in this module:

1 - SELECT: like a color filter

2 - VALUE: like a sorting filter
```js
var filterTypes = { SELECT: 0, VALUE: 1 };
```

Next, the helpers of this module will be reviewed:

### getUrlParameter

This function returns the specified parameters from the URL, which is the initialization described earlier.
```js
  if (tools.getUrlParameter('tag')) {
            var tag_value = tools.getUrlParameter('tag');
            var tag_filter = getTagValue(4);
            var tag_key = getTagValue(5);
            var tags = {tag_value: tag_value, tag_filter: tag_filter, tag_key: tag_key};
            localStorage.setItem('tagItems', JSON.stringify(tags));
            window.history.replaceState(null, null, "?tag=" + tag_value);
        }
```
### getTagValue

The `getTagValue` function takes a value from the input; for example,  number 4.
First, splits the page's location with a slash and return the 4th part of the URL.
```js
 function getTagValue(part) {
            var pathArray = window.location.href.split('/');
            var secondLevelLocation = pathArray[part];
            return decodeURIComponent(secondLevelLocation);
        }
```
### startLoading

This function first sets fetchLock to true and activates `loadingContainer` and `loadingProtectorLayer`. As mentioned, when a filter is selected, the `loadingContainer` will be displayed, and another filter cannot be chosen because `loadingProtectorLayer` is activated at the same time.
```js
  function startLoading() {
            fetchLock = true;
            if (loadingContainer.length) {
                loadingContainer.addClass('is-loading');
                loadingProtectorLayer.addClass('is-loading');
            }
        }
```
### endLoading
Six hundred milliseconds after the endLoading function is called, it sets `fetchLock` to false. That is, the lock is opened, and the layers that were activated in the previous step are deactivated.
```js
 function endLoading() {
            fetchLock = false;
            if (loadingContainer.length) {
                loadingContainer.removeClass('is-loading');
                setTimeout(function () {
                    loadingProtectorLayer.removeClass('is-loading');
                }, 600);
            }
        }
```
### generateValue
This function takes a filter from the input. If this filter has a value and the value length is greater than zero, it returns that value.
```js
  function generateValue(filter) {
            if (filter && filter.value && filter.value.length)
                return filter.value;
            return false;
        }
```
The **updateHash** function displays the selected filters in the URL. This function will be explained with an example: 

if you look at the URL in the image below, you will notice that the selected filters are after the # sign.

![FilterServiceURL.png](/FilterServiceURL.png)

When a filter is selected, the `updateHash` function is called. This function foreach on the selected filters and calls the `generateValue` function, which defines the value of each filter. Finally, it appends to the hash of the final result and connects the key and value with a slash.

### updateHash
```js
function updateHash(type = 'update') {
            var hashResult = '';
            _.each(filters, function (filter, key) {
                var value = generateValue(filter);
                if (value) {
                    hashResult += '/' + key + '/' + value;
                }
            });
            hashResult += '/';
            // check if type hash remove and tag param is isset
            if (tools.getUrlParameter('tag') && type === 'remove')
                window.history.replaceState(null, null, window.location.href.split('?')[0]);
            window.location.hash = decodeURIComponent(hashResult);
        }
```

### updateValueIds

This function iterates over all the filters. If the type was selected and the value was undefined, it foreach on the values and pushes to the `valueIds`. If the color filter was selected, it sets the `valueIds` equal to the color, and otherwise, it pushes the `valueIds` to the `selectedValues`.

```js
     function updateValueIds() {
            selectedValues = [];
            _.each(filters, function (filter, key) {
                if (filter.type === filterTypes.SELECT) {
                    if (typeof filter.value !== 'undefined') {
                        var valueIds = [];
                        _.each(filter.value, function (valueKey, index) {
                            valueIds.push(filter.options[valueKey]);
                        });
                        if (filter.is_colors) {
                            colors = valueIds;
                        } else
                            selectedValues.push(valueIds);
                    }
                }
            });

            if (tools.getUrlParameter('tag')) {
                updateTags();
            }
        }
```
### updateTags

This function shows the tags that are selected in the filter Tags Container. For example, if blue and red colors were selected in the filters, if the blue color is deselected, this blue color tag will be removed from the container tag.
```js
function updateTags() {
            console.log(filterTagsContainer);
            if (filterTagsContainer.length !== 0) {
                filterTagsContainer.html('');

                _.each(filters, function (filter, key) {
                    var value = generateValue(filter);

                    if (value && filter.type === filterTypes.SELECT) {
                        _.each(value, function (alias, index) {
                            var newOptionTag = jQuery(template.filterOptionTag({
                                "alias": alias,
                            }));

                            newOptionTag.find('.remove-filter-option-tag').on('click', function (event) {
                                event.preventDefault();

                                filter.value.splice(filter.value.indexOf(alias), 1);
                                newOptionTag.remove();
                                updateHash('remove');
                                updateValueIds();
                                loadFirst();
                                if (!filter.tag)
                                    _.each(filter.elements, function (element) {
                                        element.trigger('filter:update_value', [filter.value]);
                                    });

                                return false;
                            });

                            filterTagsContainer.append(newOptionTag);
                        });
                    }
                });
            }
        }
```
### fetchResult

Whenever any filter items on the page are selected or cleared, this function shows the output in the result container. For example, if the red color is selected, the container tag and URL will be updated and displayed in the container result.

Now the function is checked.

First, if something has not been loaded, a request is made to the server and takes the data.
Or it has been loaded, and the next page was displayed again according to the filter in the result.
First, if nothing is being loaded, the `startLoading` function is called, gets the `apiUrl`, and `next_page` is equal to one. If other pages have been loaded, `next_page` is incremented by one.


```js
   function fetchResult() {
            if (loadedResult === null || loadedResult.next_page_url !== null)
                if (!fetchLock) {
                    startLoading();
                    var apiUrl = settings.product.filter.url.get();
                    var next_page = 1;
                    if (loadedResult !== null && loadedResult.next_page_url !== null) {
                        var current_page = parseInt(loadedResult.current_page);
                        next_page = current_page + 1;
                    }
```
The value from 0 to 999999999 gives the price range and then checks whether the price range is selected. If it is selected, the price changes to the selected range. Otherwise, the same defined range is considered.
```js

                    var priceRange = ["0", "999999999"];
                    if ("price-range" in filters &&
                        typeof filters["price-range"].value !== "undefined") {
                        priceRange = filters["price-range"].value;
                    }
```
Then it sorts the products based on the desc priority. Otherwise, it updates the selected values.

```js

                    var sortDefaultData = (window.siteEnv.SITE_DEFAULT_PRODUCT_SORT || "priority:desc").split(":");
                    var sortData = {
                        field: sortDefaultData[0],
                        method: sortDefaultData[1]
                    };

                    if ("sort" in filters &&
                        typeof filters["sort"].value !== "undefined") {
                        sortData = filters["sort"].options[filters["sort"].value];
                    }


                    // check set tag search
                    if (tools.getUrlParameter('tag') && selectedValues.length === 0) {
                        updateValueIds();
                    }

```
And at the end, an Ajax request is made, the required data is filled, and it gives an output. The `pushProducts` function is called with the Ajax output; that is, it displays the products that were fetched on the page.
```js
                    jQuery.ajax({
                        url: apiUrl,
                        type: settings.product.filter.url.method.get(),
                        data: {
                            directory_id: currentDirectoryId,
                            discount: discount,
                            query: query,
                            sort: sortData,
                            price_range: priceRange,
                            colors: colors,
                            values: selectedValues,
                            page: next_page
                        }
                    }).done(function (result) {
                        if (typeof result !== 'object')
                            loadedResult = jQuery.parseJSON(result);
                        else
                            loadedResult = result;
                        setTimeout(pushProducts, settings.product.filter.delay.get());
                        setProductCounts(result.total);
                    }).fail(function (result) {
                        console.log(result);
                    });
                }
        }
```
### pushProducts

This function foreach on `loadedResult` and creates all the product boxes displayed in the result container.
```js
function pushProducts() {
            if (loadedResult.data && loadedResult.data.length) {
                _.each(loadedResult.data, function (product) {
                    product['is_cart'] = LocalCartService.isInCart(product.id);
                    product['cart_information'] = cartResult(product.id);
                    product['loaded'] = true;
                    const newProductBox = jQuery(template.productBox({product: product}));
                    newProductBox.find('.rating-content').ratingModule();
                    newProductBox.find('span.price-data').formatPrice();
                    resultContainer.append(newProductBox);
                    newProductBox.find('.ratio-box').ratioBox();
                    LocalCartService.initProductElement(newProductBox);
                });
                LocalCartService.calculateInvoice();
            } else if (loadedResult.data.length === 0) {
                resultContainer.append(noResult);
            }
            endLoading();
        }
```
### init

This function first finds the containers on the page, reads the URL hash and sets all its values, and displays the result.
```js
 init: function () {
                if (!isInitiated) {
                    filterTagsContainer = jQuery('.filter-option-tag-container');
                    productCountContainer = jQuery('#products-counts');
                    loadingContainer = jQuery('#filter-loading-container');
                    resultContainer = jQuery('#filter-result-container');
                    currentDirectoryId = resultContainer.data('directory-id');
                    query = resultContainer.data('query');
                    loadingProtectorLayer = jQuery('.loading-protector-layer');
                    discount = tools.getUrlParameter("discount");
                    if (resultContainer.length !== 0) {
                        jQuery(window).scroll(function (event) {
                            var windowObj = jQuery(this);
                            var productContainerButtom = resultContainer.offset().top + resultContainer.height();

                            var windowScrollBottom = windowObj.scrollTop() + windowObj.height();
                            if (windowScrollBottom + 500 >= productContainerButtom) {
                                loadMore();
                            }
                        });
                    }
                    isInitiated = true;
                }
            },
```

### initHash

The `initHash` function to read the URL. `init` function calls this function.
```js
 initHash: function () {
                if (!isInitiated)
                    FilterService.init();

                if (isInitiated) {
                    FilterService.translateHash();

                    jQuery(window).hashchange(function (event) {
                        FilterService.translateHash();
                    });
                }
            },
```
### translateHash

This function reads the hash and converts it into understandable content.
```js
  translateHash: function () {
                if (!isInitiated)
                    FilterService.init();

                if (isInitiated) {
                    var decodedHash = decodeURIComponent(window.location.hash);
                    var hashParts = decodedHash.split('/');
                    hashParts.splice(0, 1);
                    var recordValid = false;
                    var key = '';
                    if (hashParts.length % 2 !== 0) {
                        _.each(hashParts, function (part, index) {
                            if (index % 2 === 0) {
                                recordValid = filters.hasOwnProperty(part);
                                key = part;
                            } else if (recordValid) {
                                if (part.length > 0) {
                                    var selectedValue = part.split(',');
                                    var filter = _.clone(filters[key]);
                                    if (typeof filter.translateData === "function")
                                        selectedValue = filter.translateData(selectedValue);
                                    _.each(filter.elements, function (element, index) {
                                        if (!filter.value || JSON.stringify(selectedValue) !== JSON.stringify(filter.value)) {
                                            element.trigger('filter:update_value', [selectedValue]);
                                            FilterService.updateValue(key, selectedValue, false);
                                        }
                                    });
                                }
                            }
                        });
                    }
                }
            },
```









