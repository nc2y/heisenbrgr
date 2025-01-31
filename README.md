
<!-- README.md is generated from README.Rmd. Please edit that file -->
heisenbrgr
==========

This is a package to match accounts on anonymous marketplaces, to figure out which of them belong to the same sellers. This package is used in the paper

Xiao Hui Tai, Kyle Soska and Nicolas Christin. Adversarial Matching of Dark Net Market Vendor Accounts. 25th ACM SIGKDD International Conference on Knowledge Discovery & Data Mining (KDD), 2019.

Source code for the paper is available at <https://github.com/xhtai/adversarial-matching>.

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

The R files are broken down into four steps, 1. clean data, 2. generate account level information, 3. generate pairwise comparisons, 4. run model. Example code is given at the end, after the descriptions of all the steps.

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

Raw, timestamped scrapes of item listings: this should conatain columns `marketplace`, `date`, `seller_id`, `title`, `listing_description`, which can be matched to the `items` data using `marketplace`, `seller_id` and `title`. `listing_description` column should contain item listing descriptions and PGP keys, if any. The processing is analogous to the profile scrapes, with the only difference being that matching is done to items and by marketplace, ID and description

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

### Step 3: Pairwise comparisons

From all the account-level information generated in Step 2, generate pairwise similarity measures. Note that this step can also be run if not all of the above data are available, we just have to be careful about which comparisons to do.

Following this above analysis, `allPairwise` should have 34 variables in total, of which 25 are pairwise similarities that can be used as features for classification.

### Step 4: Model

This step uses random forests and hierarchical clustering to generate predictions for all pairs, then clusters of accounts that belong to the same seller.

### Example Code

``` r
load("/home/xtai/Desktop/tmp_9-5/testData/feedback_alphabay.Rdata")
load("/home/xtai/Desktop/tmp_9-5/testData/items_alphabay.Rdata")
load("/home/xtai/Desktop/tmp_9-5/testData/users_alphabay.Rdata")
load("/home/xtai/Desktop/tmp_9-5/testData/parsedUsers.Rdata")
load("/home/xtai/Desktop/tmp_9-5/testData/parsedItems.Rdata")

#### Step 1
outStep1 <- runStep1(feedback, items, users, parsedUsers, parsedItems)

#### Step 2
outStep2 <- runStep2(outStep1$feedback, outStep1$items, outStep1$users, outStep1$profileClean, outStep1$descriptionClean, outStep1$PGPclean, "2012-02-27", "2017-05-25")

#### Step 3
allPairwise <- runStep3(outStep2$final, outStep2$profileTokens, outStep2$titleTokens, outStep2$descriptionTokens, outStep2$inventory, outStep2$PGPlist)

#### Step 4
# this subset of the data happened to have no PGP matches, so we have to insert them for demonstrative purposes
set.seed(0)
tmp <- sample(1:nrow(allPairwise), size = .01*nrow(allPairwise), replace = FALSE)
allPairwise$PGPmatched[tmp] <- 1
##

results <- predictRF(allPairwise, c(9:17, 19:34))
clustResults <- hierarchical(rbind(subset(results$out, select = -PGPmatched), results$outTest), outStep2$final)
```

Example model fit
-----------------

We encourage users to follow the above steps to train a model that is suitable for their own purposes. If this is not an option, the package contains an example model that has been pre-trained using accounts that had at least one feedback received in August 2014. These data were collected by Nicolas Christin's group and a large subset is available from the IMPACT portal (<https://impactcybertrust.org>); the full data will be released shortly. There are a total of 2395 accounts from the following marketplaces: Agora, Alphabay, Black Market Reloaded, Dream, Evolution, Pandora, Silk Road 1, Silk Road 2 and Traderoute. 2135 of these accounts had posted at least one PGP key.

There are 2,278,045 pairwise comparisons, of which 610 are matches. Pairwise features were produced using the procedure described above for the time period 2011-05-22 to 2018-08-22.

The model is stored as a `randomForest` object in `exampleFit`. To generate predictions, users need to have 25 pairwise similarities as input: idDist, sameMarket, profileJaccard, titleJaccard, descriptionJaccard, catJaccard, catDosageJaccard, catUnitJaccard, catDosageUnitJaccard, diff\_numListingsWithFeedback, diff\_totalFeedback, diff\_dailyFrac, diff\_daysActive, diff\_diversity, diff\_meanPriceSold, diff\_medianPriceSold, diff\_minPriceSold, diff\_maxPriceSold, diff\_priceRange, diff\_numDescriptionTokens, diff\_numTitleTokens, diff\_numProfileTokens, diffSalesDates, salesOverlap, totalSalesDays. These are generated in Step 3 as described above. Say these are stored in `mySimilarities`. Predictions can then be generated as follows:

``` r
testPreds <- predict(exampleFit, mySimilarities, type = "vote")[, 2]
```

For reference: structure of data sets
-------------------------------------

``` r
str(feedback)
# 'data.frame': 307 obs. of  6 variables:
#  $ hash_str        : chr  "6a8837a3b16e29bc2252da23ceeaa4b3" "42a8abc26efb1fd65315cf5ee74b823b" "0ad07bcae6f99baf2d3b91562bd03f1d" "0a2d908d6baa04f444f001ac191ac852" ...
#  $ marketplace     : chr  "Evolution" "Alphabay" "Alphabay" "Alphabay" ...
#  $ item_hash       : chr  "00a4f9648817b79f8c01b12e2189cff4" "bde43eed0c58a595e9d65f46f2afad07" "b639970db74768d7c0b97f5b1ad74c18" "efe48ecd6e568d4749db5bd7648656ef" ...
#  $ date            : chr  "2014-11-28" "2016-10-18" "2015-06-11" "2015-11-09" ...
#  $ order_amount_usd: num  239.7 51.3 45 15 142.9 ...
#  $ vendor_hash     : chr  "037bf6237a616e6a324ec8716f6545f3" "5350c117fc60ecd4a1e3486739d566d8" "22425681f405e39035d37e6979cfadc3" "d44f012b7422868ee8eb9d55273c59e2" ...

str(users)
# 'data.frame': 21 obs. of  4 variables:
#  $ hash_str   : chr  "3844c93733cc636600cc611b6feb1460" "5505c601be9524867e517c8cbaeac775" "1d572c72bfaba7966486ed883535b18c" "e8e3a4580c7c7c7c81e20f3afd3a9d2d" ...
#  $ marketplace: chr  "Silk Road 1" "Silk Road 1" "Alphabay" "Alphabay" ...
#  $ id         : chr  "ozexpress" "captainpicard" "SpaceyElf" "SweetShop" ...
#  $ diversity  : num  0 0.1 0 0 0.04 0.53 0.12 0 0.18 0.11 ...

str(items)
# 'data.frame': 215 obs. of  8 variables:
#  $ hash_str   : chr  "2998492f78dcdea23a5f244bee034c75" "2840aeb178ff07d8d0b3e9075108ce55" "725825deb494da5af100240dc5ac5b40" "994e71105dac41b7db75ad72e0c76cfd" ...
#  $ title      : chr  "LSD 150UG" "|100%| DHL Packstation - inkl.Goldkarte & AnoSim ( ready to Use ) ab jetzt nur noch anonym einkaufen !|100% READY TO USE|" "3.5G (8th) Silver Haze *Grade A Dispensary Quality*" "DRAW FOR 1oz AAA+ Super Silver Haze - Anarcho47" ...
#  $ prediction : chr  "Psychedelics" "Misc" "Cannabis" "Cannabis" ...
#  $ dosage     : chr  "150ug" NA "3.5g" "1oz" ...
#  $ unit       : num  NA NA NA NA NA 300 NA NA NA NA ...
#  $ marketplace: chr  "Silk Road 1" "Alphabay" "Evolution" "Silk Road 1" ...
#  $ vendor     : chr  "operatorplease" "hrruin" "BudLust" "Anarcho47Lotto" ...
#  $ vendor_hash: chr  "d072a9adf44fdd2cd7d63439191c9bce" "f32a0b019a52a78236098f3a18fc6fe0" "cb0cb3dd0ec95f4469b5fad09172a091" "d993257d800984a8c04cb9fd00a1242c" ...

str(profileClean)
# 'data.frame': 47 obs. of  3 variables:
#  $ date        : num  1.39e+09 1.43e+09 1.48e+09 1.47e+09 1.49e+09 ...
#  $ vendor_hash : chr  "0315257f22effb6fc44a59fe9587e5d9" "09a9d9872484cf6679201723136de59b" "09a9d9872484cf6679201723136de59b" "a8fb37c90d8101e3556cbe4293c3be46" ...
#  $ profileClean: chr  "\n            ----------------------------------------------------------------------------------------------Wel"| __truncated__ "More About MeICQNo information AOLNo information JabberNo information WebsiteNo information No description -----" "More About MeICQNo information AOLNo information JabberNo information WebsiteNo information Any use of the info"| __truncated__ "More About MeICQNo information AOLNo information JabberNo information WebsiteNo information 13/07/16 update:\nI"| __truncated__ ...

str(descriptionClean)
# 'data.frame': 31 obs. of  4 variables:
#  $ date            : num  1.49e+09 1.45e+09 1.45e+09 1.48e+09 1.45e+09 ...
#  $ item_hash       : chr  "58135a6b917a69097cfc52b9a2848a1b" "9c1359b45718896aa401d32f9cf23717" "c9258e911a2f1d7491e8f55f3e4a221d" "26e6d627e8e4ed72ee936c993d446d12" ...
#  $ vendor_hash     : chr  "b7815ec496f6cbb0070009110e82dc0c" "c0e6838a6d2159c167484481c11d5a1f" "07ae9730df608d3bbd97576d8a603c2d" "e9f963eb74b2d2407d71a66e65cbb3a0" ...
#  $ descriptionClean: chr  "A great guide that teaches how to get refunded for clothes you have purchased off the internet. The same princi"| __truncated__ "Shipping:\nWill ship directly from India to anywhere in the world.\nSale directly from the factory (Factory pri"| __truncated__ "This listing is for 1 Adderall IR 10mg tablets (currently stocking cor pharma generics). In order for this to s"| __truncated__ "WE WANT TO INTRODUCE OUR LATEST BATCH STRAIGHT FOR COLOMBIA, THE BEST COCAINE WE HAVE SEEN IN THE PAST 30 YEARS"| __truncated__ ...

str(PGPclean)
# 'data.frame': 47 obs. of  3 variables:
#  $ date       : num  1.39e+09 1.42e+09 1.43e+09 1.48e+09 1.47e+09 ...
#  $ vendor_hash: chr  "0315257f22effb6fc44a59fe9587e5d9" "513906f2f1c16ad45d683cfc8cc5e16d" "09a9d9872484cf6679201723136de59b" "09a9d9872484cf6679201723136de59b" ...
#  $ PGPclean   : chr  "mQENBFFfhX8BCADMTWRpU5VvZpmRkaEE4D80g0LT8hz/nC25SyXWoD4HEjKaxsYt0Pu5J4hoZ8rp3Ud5W4b99Qgu+vQFXl+hbHVeRiOVl+wNArB"| __truncated__ "mQENBFRboSoBCADD++mvzsVi82aiOCLJBLvaItLQg2u6Ax5CxdYa/9fs07BkYwtEKu35+HHkvN5C885wDtktrQdSke11eV2CBvrjCb8e2wi1uHu"| __truncated__ "mQENBFUfCpcBCAC983ng8ejT41/2qEAJWQ7IUf3bgSHANQj9YKE3ciG0G0CXtufETQG4XvKzgDJJowm6I5ylWQmPYvDBPsS4UCKjKwJpEpxtCCS"| __truncated__ "mQENBFUfCpcBCAC983ng8ejT41/2qEAJWQ7IUf3bgSHANQj9YKE3ciG0G0CXtufETQG4XvKzgDJJowm6I5ylWQmPYvDBPsS4UCKjKwJpEpxtCCS"| __truncated__ ...
```

License
-------

The `heisenbrgr` package is licensed under GPLv3 (<http://www.gnu.org/licenses/gpl.html>).
