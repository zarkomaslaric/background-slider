# YelpCamp Review System

### Campgrounds index page:
![Review System Screenshot 1](https://i.imgur.com/uRIB9hr.png)

### Campground show page:
![Review System Screenshot 2](https://i.imgur.com/jMvZX1s.png)

This tutorial is based on technologies that we've learned in the course - we will integrate a review system where users can give 1-5 star ratings to individual campgrounds.

### 1) The Review Model - see the code here: [models/review.js](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/models/review.js)

First thing I focused on is the Review model that we will use for our new feature. I gave it a **rating** required field, which is an integer between 1 to 5, corresponding to the number of stars given to a campground. Also, there is a **text** field so a user can voice their opinion and explain why they gave a certain star rating.

We will also associate the author id/username like we did in the original YelpCamp application (Comment model). Also, to make things more flexible, we will be saving an ObjectId reference to the campground for which the review is created.

Finally, we pass a second parameter to **new mongoose.Schema** where we set timestamps to true, which is going to automatically give us **createdAt** and **updatedAt** fields for each review entry in the database.

### 2) Updated Campground Model - see the code here: [models/campground.js](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/models/campground.js)

To make our campground model support the new review feature, we will be adding the **reviews** ObjectId references array and **rating** which will hold the average rating for the selected campground, based on all user reviews.

```
    reviews: [
        {
            type: mongoose.Schema.Types.ObjectId,
            ref: "Review"
        }
    ],
    rating: {
        type: Number,
        default: 0
    }
```

### 3) Review Routes - see the code here: [routes/reviews.js](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/routes/reviews.js)

- Initially, we require everything necessary in our routes/review.js file:

```
var express = require("express");
var router = express.Router({mergeParams: true});
var Campground = require("../models/campground");
var Review = require("../models/review");
var middleware = require("../middleware");
```

- **Reviews - INDEX route**

```
// Reviews Index
router.get("/", function (req, res) {
    Campground.findById(req.params.id).populate({
        path: "reviews",
        options: {sort: {createdAt: -1}} // sorting the populated reviews array to show the latest first
    }).exec(function (err, campground) {
        if (err || !campground) {
            req.flash("error", err.message);
            return res.redirect("back");
        }
        res.render("reviews/index", {campground: campground});
    });
});
```

Basically, the idea is that the campground show page is going to show only 5 latest campground reviews, and that there is going to be a link to see all the reviews - it will lead to the reviews index route which we define here. The route will first find the campground in question, populate the ObjectId references in the reviews field and sort them by the createdAt date (newest first).

- **Reviews - NEW route**

```
// Reviews New
router.get("/new", middleware.isLoggedIn, middleware.checkReviewExistence, function (req, res) {
    // middleware.checkReviewExistence checks if a user already reviewed the campground, only one review per user is allowed
    Campground.findById(req.params.id, function (err, campground) {
        if (err) {
            req.flash("error", err.message);
            return res.redirect("back");
        }
        res.render("reviews/new", {campground: campground});

    });
});
```

This is going to be a GET route which will render the new review page. We attach two middleware functions in the chain, **middleware.isLoggedIn** to check if the visitor is authenticated like we've seen before, and a new middleware function **middleware.checkReviewExistence** which also checks if the user already reviewed the campground, since we only want to allow 1 single review per user. We will go over that middleware function later below.

- **Reviews - CREATE route**

```
// Reviews Create
router.post("/", middleware.isLoggedIn, middleware.checkReviewExistence, function (req, res) {
    //lookup campground using ID
    Campground.findById(req.params.id).populate("reviews").exec(function (err, campground) {
        if (err) {
            req.flash("error", err.message);
            return res.redirect("back");
        }
        Review.create(req.body.review, function (err, review) {
            if (err) {
                req.flash("error", err.message);
                return res.redirect("back");
            }
            //add author username/id and associated campground to the review
            review.author.id = req.user._id;
            review.author.username = req.user.username;
            review.campground = campground;
            //save review
            review.save();
            campground.reviews.push(review);
            // calculate the new average review for the campground
            campground.rating = calculateAverage(campground.reviews);
            //save campground
            campground.save();
            req.flash("success", "Your review has been successfully added.");
            res.redirect('/campgrounds/' + campground._id);
        });
    });
});
```

A POST route that is designed to accept the submitted form from the new page and create a new review entry in the database (we again check if user already reviewed the campground). When creating a review, we also need to recalculate and update the average campground rating, so we are also using Campground.findById() to find the campground.

Most of the logic is similar to the comments POST route that we are all familiar with, with the addition of calling the **calculateAverage** function which updates the campground.rating field (we define it at the bottom of the routes file):

```
function calculateAverage(reviews) {
    if (reviews.length === 0) {
        return 0;
    }
    var sum = 0;
    reviews.forEach(function (element) {
        sum += element.rating;
    });
    return sum / reviews.length;
}
```

It takes the array of populated reviews, then calculates an average rating and returns it from the function, which we then assign to **campground.rating** in our routes.

- **Reviews - EDIT route**

```
// Reviews Edit
router.get("/:review_id/edit", middleware.checkReviewOwnership, function (req, res) {
    Review.findById(req.params.review_id, function (err, foundReview) {
        if (err) {
            req.flash("error", err.message);
            return res.redirect("back");
        }
        res.render("reviews/edit", {campground_id: req.params.id, review: foundReview});
    });
});
```

Review GET route for the edit page, very similar to the comment edit route - we check the review ownership and allow access if a user is indeed the author of the specific review.

- **Reviews - UPDATE route**

```
// Reviews Update
router.put("/:review_id", middleware.checkReviewOwnership, function (req, res) {
    Review.findByIdAndUpdate(req.params.review_id, req.body.review, {new: true}, function (err, updatedReview) {
        if (err) {
            req.flash("error", err.message);
            return res.redirect("back");
        }
        Campground.findById(req.params.id).populate("reviews").exec(function (err, campground) {
            if (err) {
                req.flash("error", err.message);
                return res.redirect("back");
            }
            // recalculate campground average
            campground.rating = calculateAverage(campground.reviews);
            //save changes
            campground.save();
            req.flash("success", "Your review was successfully edited.");
            res.redirect('/campgrounds/' + campground._id);
        });
    });
});
```

The PUT route where we update our existing review. We first find it and submit the modifications, then find the related campground in question to calculate the new average, again using our custom **calculateAverage** function and assigning the result to **campground.rating**.

- **Reviews - DELETE route**

```
// Reviews Delete
router.delete("/:review_id", middleware.checkReviewOwnership, function (req, res) {
    Review.findByIdAndRemove(req.params.review_id, function (err) {
        if (err) {
            req.flash("error", err.message);
            return res.redirect("back");
        }
        Campground.findByIdAndUpdate(req.params.id, {$pull: {reviews: req.params.review_id}}, {new: true}).populate("reviews").exec(function (err, campground) {
            if (err) {
                req.flash("error", err.message);
                return res.redirect("back");
            }
            // recalculate campground average
            campground.rating = calculateAverage(campground.reviews);
            //save changes
            campground.save();
            req.flash("success", "Your review was deleted successfully.");
            res.redirect("/campgrounds/" + req.params.id);
        });
    });
});
```

Finally, we get to the DELETE route which gets trigerred if a user decides to delete their campground review. We also check the ownership first, then delete the campground if the check passes. Again, we need to recalculate the average campground rating because of this change, so we find the campground, use **$pull** to remove the deleted ObjectId review reference from the campground's **reviews** array and then call **calculateAverage** again to update the current rating.

At the bottom, we export the router object with `module.exports = router;`

### 4) New middleware functions for the Review System - see the code here: [middleware/index.js](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/middleware/index.js)

Make sure to require the Review model up top, in the middleware/index.js file:

`var Review = require("../models/review");`

As we already went over above, we will going to be adding 2 specific middleware functions for the review routes:

```
middlewareObj.checkReviewOwnership = function(req, res, next) {
    if(req.isAuthenticated()){
        Review.findById(req.params.review_id, function(err, foundReview){
            if(err || !foundReview){
                res.redirect("back");
            }  else {
                // does user own the comment?
                if(foundReview.author.id.equals(req.user._id)) {
                    next();
                } else {
                    req.flash("error", "You don't have permission to do that");
                    res.redirect("back");
                }
            }
        });
    } else {
        req.flash("error", "You need to be logged in to do that");
        res.redirect("back");
    }
};

middlewareObj.checkReviewExistence = function (req, res, next) {
    if (req.isAuthenticated()) {
        Campground.findById(req.params.id).populate("reviews").exec(function (err, foundCampground) {
            if (err || !foundCampground) {
                req.flash("error", "Campground not found.");
                res.redirect("back");
            } else {
                // check if req.user._id exists in foundCampground.reviews
                var foundUserReview = foundCampground.reviews.some(function (review) {
                    return review.author.id.equals(req.user._id);
                });
                if (foundUserReview) {
                    req.flash("error", "You already wrote a review.");
                    return res.redirect("/campgrounds/" + foundCampground._id);
                }
                // if the review was not found, go to the next middleware
                next();
            }
        });
    } else {
        req.flash("error", "You need to login first.");
        res.redirect("back");
    }
};
```

**checkReviewOwnership** is something that we've already seen for both campgrounds and comments routes, but **checkReviewExistence** is new - it checks if the user already reviewed the campground and disallows further actions if they did.

We find the campground first, then populate the **reviews** ObjectId references field and check the each review author contained in that array with the id of the currently logged in user.

To achieve that efficiently, we use the [some()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some) array method which will return true if any element of the array matches the check that we implement in its callback function (meaning a review with the currently logged in user was found). If none match the condition, it returns false and we know the user didn't already review the particular campground. We save the boolean result of some() to the **foundUserReview** variable which we use in the if statement after.

### 5) Updated campground routes - see the code here: [routes/campgrounds.js](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/routes/campgrounds.js)

Make sure to require the Review model up top, in the routes/campgrounds.js file:

`var Review = require("../models/review");`

- One of the main updates happens to the show route, where we chain another populate() method for the **reviews** field from the schema, which we also sort by the createdAt date (newest first):

```
// SHOW - shows more info about one campground
router.get("/:id", function (req, res) {
    //find the campground with provided ID
    Campground.findById(req.params.id).populate("comments").populate({
        path: "reviews",
        options: {sort: {createdAt: -1}}
    }).exec(function (err, foundCampground) {
        if (err) {
            console.log(err);
        } else {
            //render show template with that campground
            res.render("campgrounds/show", {campground: foundCampground});
        }
    });
});
```
- As a security measure, we add `delete req.body.campground.rating;` in the campground update (PUT) route to protect the campground.rating field from manipulation, since we are passing the req.body.campground object to the Campground.findByIdAndUpdate() method.

- As a bonus, I added logic to the campground DELETE route which will remove all the associated Comment and Review documents from the database when we delete the campground. We use the **$in** operator which finds all Comment and Review database entries which have ids contained in campground.comments and campground.reviews, and deletes them along with the associated campground that is getting removed. This is not crucial for the functionality to work, but we add it to clean up the database so there are no leftover unassociated comments or reviews in the database after the related campgroud gets deleted:

```
// DESTROY CAMPGROUND ROUTE
router.delete("/:id", middleware.checkCampgroundOwnership, function (req, res) {
    Campground.findById(req.params.id, function (err, campground) {
        if (err) {
            res.redirect("/campgrounds");
        } else {
            // deletes all comments associated with the campground
            Comment.remove({"_id": {$in: campground.comments}}, function (err) {
                if (err) {
                    console.log(err);
                    return res.redirect("/campgrounds");
                }
                // deletes all reviews associated with the campground
                Review.remove({"_id": {$in: campground.reviews}}, function (err) {
                    if (err) {
                        console.log(err);
                        return res.redirect("/campgrounds");
                    }
                    //  delete the campground
                    campground.remove();
                    req.flash("success", "Campground deleted successfully!");
                    res.redirect("/campgrounds");
                });
            });
        }
    });
});
```

Ian covers this topic in more depth in a video tutorial on his YouTube channel: [Cascade Delete with MongoDB
](https://www.youtube.com/watch?v=5iz69Wq_77k)

### 6) Updated app.js - see the code here: [app.js](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/app.js)

We need to slightly alter the app.js file, to require the new routes.js file in our main application:

```
var commentRoutes    = require("./routes/comments"),
    reviewRoutes     = require("./routes/reviews"),
    campgroundRoutes = require("./routes/campgrounds"),
    indexRoutes      = require("./routes/index")
```
Also, later in the app.js code, we add a line to use the reviewRoutes next to the other, existing routes:

```
app.use("/", indexRoutes);
app.use("/campgrounds", campgroundRoutes);
app.use("/campgrounds/:id/comments", commentRoutes);
app.use("/campgrounds/:id/reviews", reviewRoutes);
```

### 7) New EJS views for the Review System: [views/reviews/](https://github.com/zarkomaslaric/yelpcamp-review-system/tree/master/views/reviews)

- **reviews/new.ejs** is going to contain the form which a user needs to fill out to submit a new review. The first input is going to be a star rating radio button selection, where a user can choose from 1 to 5 stars. To keep the code DRY as possible, we use a neat .repeat() string method trick so we don't have to duplicate our Font-Awesome star icon code all the time. A user can also leave a text review in addition to the star rating.

**A side note:** We have to add the Font Awesome CDN link in our [views/partials/header.ejs](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/views/partials/header.ejs)

- **reviews/edit.ejs** has the form where we edit the review, which works on a similar pattern. One specific thing is that we use EJS logic to check the current **review.rating** value and add the **checked** attribute to the radio button (input element) based on that, so the current star rating is preselected when the page loads.

- **reviews/index.ejs** is our reviews index page, where we list all the reviews for a specific campground.

Check the code here: [views/reviews/index.ejs](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/views/reviews/index.ejs)

We utilize the EJS logic to check if campground already has ratings or not, and give the appropriate output according to that. We use the **.toFixed(2)** method to round the average campground to two decimals. Also, you will see that the .repeat() method is again to make sure we output the correct number of orange stars and black stars, based on the campground rating.

**Another side note:** We have to add the **.checked** CSS class to color the star orange in our [public/stylesheets/main.css](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/public/stylesheets/main.css)

We also use the .some() method again to disable the 'Write a New Review' button if the user already created a review for the campground before.

### 8) Changes to campgrounds EJS  views: [views/campgrounds/](https://github.com/zarkomaslaric/yelpcamp-review-system/tree/master/views/campgrounds)

- **campgrounds/index.ejs**: [views/campgrounds/index.ejs](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/views/campgrounds/index.ejs)

In the campgrounds/index.ejs file we will be adding similar logic like we did in the 
reviews index page, where we want to check **campground.rating** to see if the campground has any reviews. If it does, we output the star icons, and if it doesn't we just print a small message indicating that there are no reviews for the campground.

- **campgrounds/index.ejs**: [views/campgrounds/show.ejs](https://github.com/zarkomaslaric/yelpcamp-review-system/blob/master/views/campgrounds/show.ejs)

In the campground show page, we also repeat a similar logic like in the reviews index page (and the campgrounds index page above) to print the correct star rating of the campground.

Then, we have a section to print the latest reviews for the campground, using slice to only get the 5 latest ones of the campgrounds.reviews array - `<% campground.reviews.slice(0, 5).forEach(function(review){ %>`

Other logic is very similar to what we've already seen in the reviews index page.

# Final words

I hope you'll like this extra feature for YelpCamp! Make sure to check the full repository to review all code changes that needed to be done in order to implement it to our application: [https://github.com/zarkomaslaric/yelpcamp-review-system](https://github.com/zarkomaslaric/yelpcamp-review-system)

Also, a lot of credits go to Ian Schoonover (check his website [devsprout.io](https://www.devsprout.io/)) whose previous code I've used as a reference when creating this tutorial!
