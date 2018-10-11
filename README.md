
<!-- README.md is generated from README.Rmd. Please edit that file -->
heisenbrgr
==========

This is a package to match accounts on anonymous marketplaces, to figure out which of them belong to the same sellers.

Installation
------------

You can install `heisenbrgr` from github with:

``` r
# install.packages("devtools")
devtools::install_github("xhtai/heisenbrgr")
```

Data
----

We will need feedback data, item data and user data.

-   Feedback data should be unique feedback entries (timestamped), that can be matched to their corresponding item listings. We also want the amount transacted.
-   Item data should be unique items with item titles, that can be matched to the corresponding vendor account. We also want the predicted category for the item title (e.g. cannabis, benzos, digital goods, etc.).
-   User data should be unique users with IDs, marketplaces, most common category sold, and diversity of products.

Additionally, we also want raw, timestamped scrapes of profiles and item listings. This is to extract profile and item descriptions, as well as PGP keys.

Note that if you have some subset of these variables, the analysis can still be done, with some modifications. At a minimum, this is what we need:

-   Unique feedback entries (timestamped), that can be matched to their corresponding item listings
-   Unique item titles, that can be matched to the corresponding vendor account
-   Unique users with IDs and marketplaces
-   Some means of extracting PGP keys (e.g. from user profiles, item descriptions)

Steps in analysis
-----------------

The R files are broken down into four steps, 1. clean data, 2. generate account level information, 3. generate pairwise comparisons, 4. run model.

### Step 1a: Clean feedback, items and users data

Given the data as described in the Data section (above), we prepare this for further analysis. In particular,

For feedback data:

-   add `vendor_hash` to link to users data, and delete rows that don't have a match

For items data:

-   delete items that don't have any feedback (we won't be analyzing these anyway)
-   extract inventories:
    -   `category`: predicted category, same as in users data
    -   `dosage` (number + unit, e.g. "8g"): number of grams, ounces, pounds, milligrams, micrograms
    -   `unit` (numeric): number of pills, tabs, tablets, blotters, or beginning quantity in item title

For users data:

-   delete rows that have no match in feedback data (we won't be analyzing these anyway)

`Step1a_cleanData.R` does the above, producing `feedback`, `users`, `items` that can be used in Step 2. If we have raw, timestamped scrapes of profiles and item listings, we continue with the following processing.

### Step 1b: Clean raw, timestamped scrapes of profiles and item listings

Raw, timestamped scrapes of profiles: this should conatain columns `marketplace`, `date`, `id`, `profile`, which can be matched to the `users` data using `marketplace` and `id`. `Profile` column should contain profile descriptions and PGP keys, if any.

-   First remove rows with missing information, or duplicated rows
-   Match to users by marketplace and ID, and remove any non-matches
-   Extract PGP keys
-   Remove duplicates and those that clearly have no information, e.g. "\\n-----\\n", "-----"
-   make sure that at one date there is a single entry – cat the profiles (for alphabay check that PGP keys are the same) --- check: I don't think this was done

Raw, timestamped scrapes of item listings: this should conatain columns `marketplace`, `date`, `seller_id`, `title`, `listing_description`, which can be matched to the `items` data using `marketplace`, `seller_id` and `title`. `listing_description` column should contain item listing descriptions and PGP keys, if any. The processing is analogous to the profile scrapes.

-   First remove rows with missing information, or duplicated rows -- Match to items by marketplace, ID and description, and remove any non-matches
-   Extract PGP keys
-   Remove duplicates and those that clearly have no information, e.g. "\\n-----\\n", "-----"

`Step1b_cleanParsedData.R` does the above, producing `profileClean`, `descriptionClean`, `PGPclean` that can be used in Step 2. Note that additional pre-processing may be desirable, for example there is filler text in some descriptions (e.g. in SR2: "Overall Average ... description"), and it would be beneficial to remove these.

### Step 2: Generating account level information

In this step we generate account level information given some time period of interest. In order to do so we would need the following cleaned data sets, generated from Step 1. The structure of these data sets is given in the reference section below.

-   `feedback` (timestamped)
-   `users`: unique user/vendor accounts
-   `items`: unique item listings
-   `profileClean` (timestamped profiles)
-   `descriptionClean` (timestamped descriptions)
-   `PGPclean` (timestamped)

What this step does is to:

-   get accounts that have feedback in particular chosen time period
-   then get information about vendor: vendor\_hash, id, marketplace, category, diversity
-   time-specific information throughout period:
    -   number of items with feedback (from feedback)
    -   5-number summary of prices (from feedback)
    -   sales days (from feedback)
    -   number of feedbacks (+ normalize by marketplace)
    -   item titles + inventories (from items)
    -   tokenized profile, also from equal periods before and after if unavailable (from parse users)
    -   tokenized item descriptions, also from equal periods before and after if unavailable (from parse items)
    -   PGP key, also from equal periods before and after if unavailable (from parse both)

Account-level data sets created by this set of functions: `final` + list items `titleTokens`, `inventory`, `profileTokens`, `descriptionTokens`, `PGPlist`

Example code: Let time period of interest be 2014/08/01-2014/08/03.

``` r
load("/home/xtai/Desktop/markets/code/task00_everything/feedback.Rdata")
out <- infoFromFeedback(feedback, "2014-08-01", "2014-08-03")

load("/home/xtai/Desktop/markets/code/task00_everything/users.Rdata")
usersOut <- infoFromUsers(users, out$vendorHashes)

load("/home/xtai/Desktop/markets/code/task00_everything/items.Rdata")
itemsOut <- infoFromItems(items, out$itemHashes, out$vendorHashes)

load("/home/xtai/Desktop/markets/code/task00_everything/profileClean.Rdata")
profileCleanOut <- infoFromProfileClean(profileClean, out$vendorHashes, "2014-08-01", "2014-08-03")

load("/home/xtai/Desktop/markets/code/task00_everything/descriptionClean.Rdata")
descriptionCleanOut <- infoFromDescriptionClean(descriptionClean, out$vendorHashes, out$itemHashes, "2014-08-01", "2014-08-03")

load("/home/xtai/Desktop/markets/code/task00_everything/PGPclean.Rdata")
PGPout <- generatePGPs(PGPclean, out$vendorHashes, "2014-08-01", "2014-08-03")
    
###### These are the data produced
final <- cbind(out$out, usersOut[, 2:3], numTitleTokens = itemsOut$out[, 2], numProfileTokens = profileCleanOut$out[, 2], numDescriptionTokens = descriptionCleanOut$out[, 2])
itemsOut$titleTokens
itemsOut$inventory
profileCleanOut$profileTokens
descriptionCleanOut$descriptionTokens
PGPout
```

### Step 3: Pairwise comparisons

From all the account-level information generated in Step 2, generate pairwise similarity measures. Note that this step can also be run if not all of the above data are available, we just have to be careful about which comparisons to do.

The following continues the above example.

``` r
# if you don't have any of these information, don't run that line
allPairwise <- getPairs(final)
profileJaccard <- fromList(allPairwise, profileCleanOut$profileTokens, jaccardSimilarity)

titleJaccard <- fromList(allPairwise, itemsOut$titleTokens, jaccardSimilarity)

descriptionJaccard <- fromList(allPairwise, descriptionCleanOut$descriptionTokens, jaccardSimilarity)

catJaccard <- fromList(allPairwise, lapply(itemsOut$inventory, function(x) x$category), jaccardSimilarity)
catDosageJaccard <- fromList(allPairwise, lapply(itemsOut$inventory, function(x) x$catDosage), jaccardSimilarity)
catUnitJaccard <- fromList(allPairwise, lapply(itemsOut$inventory, function(x) x$catUnit), jaccardSimilarity)
catDosageUnitJaccard <- fromList(allPairwise, lapply(itemsOut$inventory, function(x) x$catDosageUnit), jaccardSimilarity)

PGPmatched <- fromList(allPairwise, PGPout, PGPmatch)

absDifferences <- absFromDataframe(allPairwise, final, c("numListingsWithFeedback", "totalFeedback", "dailyFrac", "daysActive", "diversity", "meanPriceSold", "medianPriceSold", "minPriceSold", "maxPriceSold", "priceRange", "numDescriptionTokens", "numTitleTokens", "numProfileTokens"), c(rep(0, 10), rep(1, 3)))

colNums <- which(substr(names(final), 1, 2) == "20")
salesDiffs <- fromSalesDays(allPairwise, final[, colNums])

allPairwise <- cbind(allPairwise, profileJaccard, titleJaccard, descriptionJaccard, catJaccard, catDosageJaccard, catUnitJaccard, catDosageUnitJaccard, PGPmatched, absDifferences, salesDiffs)
```

Following this above analysis, `allPairwise` should have 34 variables in total, of which 26 are pairwise similarities that can be used as features for classification.

### Step 4: Model

TODO

For reference: structure of data sets
-------------------------------------

``` r
str(feedback)
# 'data.frame': 3870168 obs. of  6 variables:
#  $ hash_str        : chr  "6658bc74e24e45c1300e69a0b59c7f8e" "43e4f3a5e5855c76c1ebc86118beb82e" "5bea028a082b6b17c6bdc8fb9c5c0616" "f27315f3ef01ebed4f1e855ccda2a3ad" ...
#  $ marketplace     : chr  "Alphabay" "Alphabay" "Alphabay" "Alphabay" ...
#  $ item_hash       : chr  "630c59284d1cf02815535cb30012e67a" "630c59284d1cf02815535cb30012e67a" "31408eabdb34db0a87b791a27b075ae6" "31408eabdb34db0a87b791a27b075ae6" ...
#  $ date            : chr  "2015-01-01" "2015-01-01" "2015-01-04" "2015-01-04" ...
#  $ order_amount_usd: num  137 137 4 4 137 ...
#  $ vendor_hash     : chr  "62fd9a95ecdc4e4632dbfb5bebd5504d" "62fd9a95ecdc4e4632dbfb5bebd5504d" "68cc671e4df4db478d3ef3c8e8987991" "68cc671e4df4db478d3ef3c8e8987991" ...

str(users)
# 'data.frame': 15596 obs. of  4 variables:
#  $ hash_str   : chr  "20b43946b247e2156de1db230a184a86" "cfbb1d986dd4746d76cda762d0ef37e8" "9b19cc878a73213280eccb208b2f9462" "c63a24e74a97fb2fe6cfabf95898f6c8" ...
#  $ marketplace: chr  "Alphabay" "Alphabay" "Alphabay" "Alphabay" ...
#  $ id         : chr  "QTwoTimes" "Hatoko" "PizzaHub" "CuriosityLabs" ...
#  $ diversity  : num  0 0 0.09 0 0 0.49 0.18 0 0 0 ...

str(items)
# 'data.frame': 228728 obs. of  8 variables:
#  $ hash_str   : chr  "94b1ec73ec88315cb1d8cc5c8f77eed7" "3970609f88e98eb3cc1562a229b2249b" "74309618d4cd9b71485e48ff75084730" "e0953794b5d5338fe4fdcb533a913a3c" ...
#  $ marketplace: chr  "Alphabay" "Alphabay" "Alphabay" "Alphabay" ...
#  $ title      : chr  "SAPIO is LIVE!!!!!!!!! SPACE DIESEL LIVE RESIN NUG RUN!!!!!!!!!!! (2 gram) BEST OF THE BEST SAPIO CUP WINNER!!!!!!! AAA GRADE" "28g PEARL COCAINE STILL ON THE BRICK! THIS IS HIGH 90'S GUYS" "20g \"Lemon Kush\"" "1 gram medical grade shatter" ...
#  $ vendor     : chr  "SapioWax" "AusKing" "CaliforniaDreams420" "DrReefer" ...
#  $ vendor_hash: chr  "bb3f06b99219bb0defe4a8d1e7227532" "3bfdb89970fb5d9e14343f2712cc9599" "9e85840fc56b6d8a18daebab41c1665b" "e8c62da88c24afe61036a073eeeb1477" ...
#  $ prediction : chr  "Cannabis" "Stimulants" "Cannabis" "Cannabis" ...
#  $ dosage     : Factor w/ 1036 levels "0.000714285714285714g",..: 623 611 449 417 276 NA 1002 78 627 NA ...
#  $ unit       : num  NA NA NA NA 1 10 5 NA 10 NA ...

str(profileClean)
# 'data.frame': 94339 obs. of  3 variables:
#  $ date        : num  1.47e+09 1.47e+09 1.47e+09 1.47e+09 1.47e+09 ...
#  $ vendor_hash : chr  "0b719c6e4a78e28079e52a4f1829b44b" "5a2bcf0e2071e45806f7d5cf9b0354ca" "7074e82df06ae71a55f278d3863c0c28" "e73e88cae18b8a53c4c21a47c99eaf86" ...
#  $ profileClean: chr  "More About MeICQNo information AOLNo information JabberNo information WebsiteNo information Welcome buyer, frie"| __truncated__ "More About MeICQNo information AOLNo information JabberNo information WebsiteNo information ==================="| __truncated__ "More About MeICQNo information AOLNo information JabberNo information WebsiteNo information I am the GREEN GIAN"| __truncated__ "More About MeICQNo information AOLNo information JabberNo information Websitehttp://grams7enufi7jmdl.onion/info"| __truncated__ ...

str(descriptionClean)
# 'data.frame': 2392596 obs. of  4 variables:
#  $ date            : num  1.47e+09 1.47e+09 1.47e+09 1.47e+09 1.47e+09 ...
#  $ item_hash       : chr  "126601368428a79b5479f248962de222" "c376cfe8b09d4c3509841e40771f2b35" "73775e1e8d93eca8f6b8f2617aae940d" "6985fdc4a4d0058089afed28da45551e" ...
#  $ vendor_hash     : chr  "a16c680e7a938115cbc57c163ef491d4" "9d724e0c742ab87e0ddcb76ad64a4097" "c4d5658385fcd360e894bb0bcb8259b3" "094e745b0eb894d1107fa02389d6131f" ...
#  $ descriptionClean: chr  "Ladies and Gentleman\n This is our DELUXE cocaine for those seeking something very high grade. For the regular "| __truncated__ "The Magic Mushrooms are in perfect condition. These Magic Mushrooms are considered A++ quality shrooms. from th"| __truncated__ "+++CGC+++ SUMMER SALE! +++ ALL ITEMS %30 OFF! +++\nNow you can get the best customer service and medical cannab"| __truncated__ "28g (1OZ) of 100% Ozzie dry buds and potent (well as potent as POG gets) hydro weed with a pretty good price to"| __truncated__ ...

str(PGPclean)
# 'data.frame': 162150 obs. of  3 variables:
#  $ date       : num  1.47e+09 1.47e+09 1.47e+09 1.47e+09 1.47e+09 ...
#  $ vendor_hash: chr  "0b719c6e4a78e28079e52a4f1829b44b" "5a2bcf0e2071e45806f7d5cf9b0354ca" "3cd4898d3d24d1d1fecf3b9f3fa04853" "7074e82df06ae71a55f278d3863c0c28" ...
#  $ PGPclean   : chr  "mQENBFFfhX8BCADMTWRpU5VvZpmRkaEE4D80g0LT8hz/nC25SyXWoD4HEjKaxsYt0Pu5J4hoZ8rp3Ud5W4b99Qgu+vQFXl+hbHVeRiOVl+wNArB"| __truncated__ "mQINBFLVvCkBEADeE/L7uBq3Xutp2EJqssjqts08Ziid/a7Sada4u+OAxjL4zYSjdSsPj21Ebao60qaejuebmFhUi39b0FvPA8j85bSuaG6OyQ+"| __truncated__ "mQENBFbmvJoBCADbBRB3B8L5mQKPO9bHSPSoT93IC+1Xwqqs0acOWm7nopgXHGneJX1LFquEOBKgcqmY3dT2rU6ocT+irkuHuqdqXKzmbjU2EJU"| __truncated__ "mQENBFZS22oBCACjujiJk0z4aiRMzmlwONUiz7USQ5KifiMRHYEbMkehuCZKUTIGHsjbrT7rNsPF/1ttFuB4oBAmJUxs0jn0EIdm0OZ/xwIGwUK"| __truncated__ ...
```

License
-------

The `heisenbrgr` package is licensed under GPLv3 (<http://www.gnu.org/licenses/gpl.html>).