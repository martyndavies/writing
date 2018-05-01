---
title: Building an image search with Algolia & Google’s Vision API
published: true
description: Build an 'auto-tagging' image search service using Algolia and Google Cloud's Vision API
tags: algolia, google vision, javascript, api
---
Being able to search through uploaded content is always useful, but the quality of the search terms is usually down to the individuals that upload the content. It’s typically their job to either describe the content as free text, or choose from pre-defined tags.

This is fine, but it’s typically an extra step to completion that delays the user experience, or the input from the user is so random (“lol doggos 😆”) that it isn’t going to make for very useful search data.

Thankfully, it’s 2018 and technology has progressed enough that there are ways to ease this situation. So, I set out to create a simple image search app that uses [Algolia’s](https://www.algolia.com) powerful indexing and search experience libraries with a selection of animal photos (just because) that are automatically classified by Google Cloud’s [Vision API](https://cloud.google.com/vision/).

![What the app looks like](https://cl.ly/2k1m0i3R2F1L/Screen%20Recording%202018-03-26%20at%2005.24%20pm.gif)

This example app allows users to upload images, which are then automatically classified (which is really just a fancy way of saying ‘tagged’) and colour profiled by the Vision API. These results are pushed to an Algolia index which then allows them to be immediately searched.

We’re going to take a look at three of the key elements of the app here, but all of the source code is [available on GitHub](https://github.com/martyndavies/cloud-vision-algolia) so you can take a look at the whole app in its full context.

## 1. Classification
The classification of the images is the most key part of this application, yet getting those results is actually quite simple thanks to the work Google’s engineers have put in to make the Cloud Vision API quick and easy to use.

After setting up our account (which comes with a free $300 usage credit) and fighting through the credentials set up process (it isn’t hard, it’s just a bit lengthier than I’d like it to be), we ended up with this piece of code for getting the information we need:

```javascript
const vision = require('@google-cloud/vision');
const imageClient = new vision.ImageAnnotatorClient();

// classifyImage() function
const classifyImage = (image, cb) => {
 
  // Use the locally stored image from the upload
  const imageToClassify = `./public/images/${image}`;

  // Ask Google Vision what it thinks this is an image of
  imageClient
  .labelDetection(imageToClassify)
  .then(results => {
    const imageLabels = results[0].labelAnnotations;

      // Also ask for the dominant colors to use as search attributes
      imageClient
      .imageProperties(imageToClassify)
      .then(results => {
        const properties = results[0].imagePropertiesAnnotation;
        const dominantColors = properties.dominantColors.colors;

        // Pass both lists back in the callback
        cb(imageLabels, dominantColors);
      })
      .catch(err => {
        console.error('Error:', err);
      })
  })
  .catch(err => {
    console.error('Error:', err);
  });
};
```

Here’s what’s happening above:

After initialising our instance of Google Cloud Vision’s Node JS library we pass it an image and ask for a list of tags in return. Then, once we have those, we ask it to also return a list of colours that are present in the image as well.

_Note: The above code is taken directly from the [example app I’ve build for you to play around with](https://github.com/martyndavies/cloud-vision-algolia), but it looks a bit long, so from this point on I’ll be using simplified versions of the code I wrote._

To start with, a simplified version of this that just logs the tags to the console would be:

```javascript
function getImageLabels(image) {
  imageClient
  .imageProperties(image)
  .then(results => {
    // The labels
    const imageLabels = results[0].labelAnnotations;
    console.log(imageLabels);
  })
}

getImageLabels('./myPicture.jpg');
```

What the API returns is an array of JSON objects that look like this (if you upload a picture of a panda in a tree 🐼):

![Said panda. In a tree. Credit: Getty Images](https://www.telegraph.co.uk/content/dam/news/2016/08/23/106598324PandawaveNEWS_trans_NvBQzQNjv4Bqeo_i_u9APj8RuoebjoAHt0k9u7HhRJvuo-ZLenGRumA.jpg?imwidth=300)

```javascript
[{
  locations: [],
  properties: [],
  mid: '/m/03bj1',
  locale: '',
  description: 'giant panda',
  score: 0.9907882809638977,
  confidence: 0,
  topicality: 0.9907882809638977,
  boundingPoly: null
}]
```

As you can see, the detail you get back is very extensive and can include location information, boundary information and even crop suggestions if you want them. For now though, we only require the `description` and the `score`  (which is how certain Google is about the image) for this app.

Now, you could pass all of this over to your Algolia index if you wanted to, especially if you were working with images that did return more data for facets like locale and locations. This would make for good search data!

We’re only going to work with labels in this demo so let’s pluck out the `score` and the `description` tag and create a new object that we’ll later pass over to Algolia for indexing:

```javascript
function reduceLabelsToObject(labels) {
  // Construct a new object with a couple of pre-defined keys
  // and a link to our uploaded image
  const algoliaData = {
    labels: [],
    upload_date: Date.now(),
    image_url: '/images/image.jpg'
  };

  // Loop through the labels and add each one to the
  // 'labels' array in the object
  labels.forEach(attribute => {
    algoliaData.labels.push({
      classification: attribute.description,
      score: attribute.score
    });
  });
}
```

## 2. Indexing
Once we have a result from the Vision API, it’s time to put that data somewhere more useful so that it can be searched. We’re going to store it in Algolia via their JavaScript SDK.

Above, we created a JavaScript object of the information we want to store, it’s called `algoliaData`, so let’s push this to our index:

First, make sure your Algolia set up is correct by loading the library, setting the API keys, specifying which index you want to look at and use and _most importantly_ which attributes users will be able to search:

```javascript
// Require the library
const algolia = require('algoliasearch');
// Init the client with your APP ID and API KEY
const client = algolia('your_app_id', 'your_admin_api_key');
// Assing which index you want to use
const index = client.initIndex('images');

// Set some settings on the index, make sure only the
// labels array is searchable
index.setSettings({
  'searchableAttributes': [
    'labels.classification'
  ]
});
```

Then push the data to the index:

```javascript
const addToAlgoliaIndex = (algoliaData) => {
  index.addObject(algoliaData, function(err, content) {
    if (err) {
	    console.log(`Error: ${err}`
    } else {
      console.log(`All good: ${content}`
    } 
  });
}
```

That’s actually everything. Algolia can index JSON in any form so your keys and values can be whatever you like. At it’s most simple, the `index.addObject()` method does everything you need to add single objects to the index quickly and easily.

At this point we’ve set up image recognition and subsequent classification (tagging) and we’ve uploaded that image information to Algolia, which now means it’s searchable.

## 3. Displaying results
The final piece of the puzzle for this app is how to display the images that are being uploaded back to the users, and allow them to be searched.

Algolia does allow us to build out a search experience using their APIs and we could make it as tuned and customised as we like. In the interest of time though, we’re going to use the excellent [InstantSearch.js]([InstantSearch.js](https://community.algolia.com/instantsearch.js/)) library they provide to create a great search experience using a series of pre-defined widgets that we can style to our liking.

### Setting up InstantSearch
You can add InstantSearch to your front end by downloading it, adding it via a package manager, or loading it from a CDN. You can check out all those installation options in the [documentation]([InstantSearch.js | Getting started](https://community.algolia.com/instantsearch.js/v1/documentation/)).

Once you’ve loaded InstantSearch.js, you can initialise it in a separate JS file, or inside a `<script>` tag:

```javascript
const search = instantsearch({
  appId: 'your_app_id',
  apiKey: 'your_api_key',
  indexName: 'images'
});

search.start();
```

### Adding a search box
…could not be simpler. We’ll use one of the [built in InstantSearch widgets]([Documentation - instantsearch.js](https://community.algolia.com/instantsearch.js/v1/documentation/#widgets)) to add this to our app.

In our HTML, after adding the InstantSearch.js files and CSS, we add:

```html
<div id=“search-box”></div>
```

Then in our JS file:

```javascript
search.addWidget(
  instantsearch.widgets.searchBox({
    container: '#search-box',
    placeholder: 'Search for images'
  })
);
```

Above, we’re adding the Search Box widget to the `search` instance and telling it to load all the elements into the `<div>` with the ID of `search-box`.

A search box is cool ’n’ all but if the results don’t have anywhere to display, it’s still quite useless. Let’s set up how we’re going to display the search results that are returned when something is typed into the search box.

Start by adding another `<div>` to your HTML to house the results:

```html
<div id=“hits></div>
```

Then in your JS file, add the Hits widget:

```javascript
search.addWidget(
  instantsearch.widgets.hits({
    container: '#hits',
    templates: {
      empty: `<p>Sorry, we couldn't find any matches.</p>`,
      item: function(hit) {
        return `
        <div class="card">
          <div class="card-image">
            <img src="${hit.image_url}" class="image">
          </div>

          <div class="card-action">
            <a href="${hit.image_url}" download="${hit.image_url}">Download</a>
          </div>
          <div class="card-footer" style="height:10px; background-color:${hit.most_dominant_color}"></div>
        </div>
      `
      }
    }
  })
);
```

Each result that Algolia returns is known as a _‘hit’_. The Hits widget allows us to specify where in our HTML these results should be displayed, as well as what they should look like.

In our example app, the hits are styled using [Materialize CSS]([Documentation - Materialize](http://materializecss.com/)) and they look like this:

![An example hit](https://cl.ly/1q2i21081a13/Image%202018-03-26%20at%203.44.03%20pm.png)

There are two templates in use in the code above. The first is what should be displayed if there are no results at all. The second is what each result should look like if there are results (hits) to display.

Each result is passed into the function as an object and you can reference any of the attributes in the HTML. As you can see from the template, we require the `image_url` attribute and the `most_dominant_color` attribute to fill out the content of our card.

## That’s it. Fin.
Through these examples you’ve seen how to do the following:
* Return classification data from Google Cloud’s Vision API by passing it an image
* Store relevant information about this image in Algolia and make it searchable
* How to add a search interface and search results to your app quickly using InstantSearch.js

If you take a look at the [full source code of the example app](https://github.com/martyndavies/cloud-vision-algolia) you’ll also get to see how the image uploading is handled using JavaScript and a library for NodeJS called [Multer](https://github.com/expressjs/multer). You’ll also see how to work with some of the dynamic components that [Materialize CSS](http://materializecss.com/) offers, such as modals and notifications.

If you have any questions about any of this then feel free to reach out to me via [GitHub](https://github.com/martyndavies/cloud-vision-algolia), or via [Twitter](https://twitter.com/martynd).