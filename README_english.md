Short link we have more or less used, the so-called short link is based on the longer original link url to generate a shorter link, visit the short link can jump to the corresponding original link, the benefits of doing so are: 1. url more beautiful; 2. easy to save and spread; 3. some website content release has a word limit, short links can save the word count.

The principle of implementing short links is very simple and can be summarized as follows.
1. generate a unique short link for each original link without duplication
2. save the original link and the corresponding short link in pairs to the database
3. when accessing the short link, the web server will redirect the target to the corresponding original link

According to the above idea, we can also implement a short link generation service by ourselves in minutes. This example uses node + express + mongodb.

# 1. initialize the project

## (1). Install the following dependencies.

**package.json:**

```
"dependencies": {
  "config": "^3.2.2", // read the project configuration
  "express": "^4.17.1", // web server
  "mongoose": "^5.6.9", // operate mongodb
  "shortid": "^2.2.14", // generate non-repeating unique Id
  "valid-url": "^1.0.9" // determine if the url format is correct
}
```

## (2). Add project configuration.

Mainly used to store MongoDB connection strings and base url for short links.

**config/default.json:**

```
{
  "mongoURI": "mongodb://localhost:27017/url-shorten-service",
  "baseUrl": "http://localhost:5000"
}
```

## (3). Add MongoDB connection method

**config/db.js:**

```
const mongoose = require('mongoose');
const config = require('config');
const db = config.get('mongoURI');

const connectDB = async () => {
  try {
    await mongoose.connect(db, {
      useNewUrlParser: true
    });
    console.log(`MongoDB Connected to: ${db}`);
  } catch (error) {
    console.error(error.message);
    process.exit(1);
  }
}

module.exports = connectDB;
```

## (4). Start express.

**index.js:**

```
const express = require('express');
const connectDB = require('. /config/db');

const app = express();

// connect to MongoDB
connectDB();

app.use(express.json({
  extended: false
}));

// Routing, to be set later
app.use('/', require('. /routes/index'));
app.use('/api/url', require('. /routes/url'));

const port = 5000;

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

# 2. Defining the database model

We need to save the original link and the corresponding short link to the database, for simplicity, we just need to save a short link code, the corresponding short link can be stitched together using the base url and the code.

**models/url.js:**

```
const mongoose = require('mongoose');

const urlSchema = new mongoose.Schema({
  urlCode: String,
  longUrl: String
});

module.exports = mongoose.model('Url', urlSchema);
```

# 3. Generate short link code

This is a key step in our implementation, the idea is: the user passes in a long link, we first use [valid-url](https://www.npmjs.com/package/valid-url) to determine whether the incoming url is legal, if not then return an error, if legal we search the database to see if there is a record of the long link. If there is, we return the record directly, if not, we generate a new record and the corresponding short link. With the help of [shortId](https://www.npmjs.com/package/shortid), we can easily generate a non-repeating unique code.

**routes/url.js:**

```
const epxress = require("express");
const router = epxress.Router();
const validUrl = require('valid-url');
const shortId = require('shortid');
const config = require('config');
const Url = require('. /models/url');

router.post('/shorten', async (req, res, next) => {
  const { longUrl } = req.body;
  if (validUrl.isUri(longUrl)) {
    try {
      let url = await Url.findOne({ longUrl });
      if (url) {
        res.json({
          shortUrl: `${config.get('baseUrl')}/${url.urlCode}`
        });
      } else {
        const urlCode = shortId.generate();
        url = new Url({
          longUrl,
          urlCode
        });
        await url.save();
        res.json({
          shortUrl: `${config.get('baseUrl')}/${urlCode}`
        });
      }
    } catch (error) {
      res.status(500).json('Server error');
    }
  } else {
    res.status(401).json('Invalid long url');
  }
});

module.exports = router;
```

# 4. Visit the short link to jump to the original link

The last step is very simple, when the user accesses the short link we generated, we look up the corresponding record based on the short link code in the url, if the corresponding record exists we use the `res.redirect` method of express to redirect the access to the original link, if it does not exist then we return an error.

**routes/index.js**

```
const epxress = require("express");
const router = epxress.Router();
const Url = require('. /models/url');

router.get('/:code', async (req, res, next) => {
  try {
    const urlCode = req.params.code;
    const url = await Url.findOne({ urlCode });
    if (url) {
      // Redirect to the original link
      res.redirect(url.longUrl);
    } else {
      res.status(404).json("No url found");
    }
  } catch (error) {
    res.status(500).json("Server error");
  }
});

module.exports = router;
```

**Test it out:**

! [send request](http://lc-PX2vd1LW.cn-n1.lcfile.com/417bbdd2c4f467425bac/1.png)

** Access the short link:**

! [clipboard.png](/img/bVbwF5x)

In this way, a simple short link generation service is completed, often in our view amazing technology in fact the principle behind and implementation is very simple, I hope this article has inspired you.

> Finally, we recommend you to use [Fundebug](https://www.fundebug.com/?utm_source=MudOnTire), a very useful bug monitoring tool ~

This article Demo address: https://github.com/MudOnTire/url-shortener-service
