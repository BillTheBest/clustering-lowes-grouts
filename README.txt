Using Clustering to Suggest Tags

I have a concrete counter top that is coloured with a thin surface layer of multiple shades of concrete dye all mixed together.  It has a wonderful earthy look to it, but I've discovered that it doest stand up to my wife swinging around heavy pots and pans!  I tried a sanded grout witht the brand name "Mapei" from Lowes Home Improvement, but it didn't really have the effect I wanted, leaving a built up area that wouldn't take the concrete sealer that makes the counter look shiny.   So I went onto Lowes's website and searched for "Mapei Unsanded Grout" (http://www.lowes.com/SearchCatalogDisplay?storeId=10151&langId=-1&catalogId=10051&N=0&newSearch=true&Ntt=mapei+unsanded+grout), and recieved back 30 results:

https://skitch.com/epugh/8yqa2/shop-mapei-unsanded-grout-at-lowes.com-search-results

While on the face of it, 30 results back sounds like a great thing, but then I scrolled down the page and was overwhelmed by all the choices.  It seemed like I had 30 differenct choices to pick from.  Go ahead, click the link and try it yourself!  

I scrolled up and down a couple of times and started realizing that I didn't have 30 choices.  I actually had TWO choices: a "Unsanded Powder Grout" and a "Preimum Unsanded Powder Grout", and they come in a variety of colors and sizes.  So here I am, frantically scrolling up and down trying to build a mental model of all my choices, when I think "ah, facets should help me".  I knew I was doing just a bit of patching, so Iw atned the smallest size possible. And I knew I wanted a rich earthy red/brown color to match the existing concrete counter top.  Unfortunantly neither "Size" nor "Color" are facet options.

So I thought, instead of trying to build a mental model of all thirty results, why don't I leverage clustering to see if I can pull out of the unstructured data some shared clusters that would act as facets?   This blog post and the accompanying code at "https://github.com/o19s/clustering-lowes-grouts" is going to walk you through how I did this.

1) Extracting the Data
I wrote a very simple Ruby script that queries Lowes.com for "mapei unsanded grout", and the parses out the results and stores them into a Solr index.  Fire up Solr by running ./start.sh.  Then run the script "ruby download_lowes_data.rb".  You may have to install some gems!

2) View the Clusters
I leveraged the builtin UI "Solritas" to expose the data about the clusters.  Browse to http://localhost:89983/solr/browse to see the results of clustering the data:

https://skitch.com/epugh/8yupa/solritas

It's not very exciting at this point.  I've told solr to cluster primarily on the title, and secondarily on the product detail bullets.  And it has accurately identified that there are a number of grouts that come in the 10 lb bag.  And that we have some premium grouts.  More useful would be those that are 10 lbs, and those that are 25 lbs however!  Although, one thing to note is that by clustering, we did successfully extract the root product title for two of the items, and dropped the color name portion.  MAPEI 10 lb. Keracolor U Premium Unsanded Grout - Lt.Almond #49 and MAPEI 10 lb. Keracolor U Premium Unsanded Grout - Chocolate #07 both have the correct root title of Keracolor U Premium Unsanded Grout.


3) Adding in Facets
I had turned on faceting on for the Item Number and Model Number fields, and set the miniuim facet count to display to 2 because I only care about item numbers or model numbers that are shared by multiple products.  While each product does have a unique Model Number, what was interesting to discover was that there are 14 products out of the 30 that share the same Item Number: 185276.  It appears that "Item Number" is potentially assigned by the manufacturer, and idenfies a single product, regardless of size or color choice, while model number is the unique SKU for that product.

https://skitch.com/epugh/8yutg/solritas

I chose that facet (http://localhost:8983/solr/browse?&q=mapei+unsanded+grout&fq=item_number_s:%22Item+%23%3A+185276%22) and all of a sudden my clusters started looking more interesting!   I have 5 clusters identified: Unsanded Powdered Grout, Keracolor U Premimum Unsanded Grout, MAPEI 10 Lbs, MAPEI 25 Lbs, and Help Contribute to LEED Certification of Projects.   The first four clusters all look really great, as I can see that I have a set of products that are 10 lbs, 25 lbs, and either the Unsanded Powder Grout or the Premium Unsanded Grout.   Even the fifth facet about LEED certification could be useful in helping me identify products that contribute towards LEED cerfitication.   One thing I noticed though was that the total unique number of clustered items didn't match the number of products returned by the faceted query.  Turns out the clustering results are returned via an AJAX call to the /clustering handler, and it has a limit of 10 rows.  So I bumped it up to 100 so I would see all the documents returned.

4) Playing with Clustering Options
The /clustering request handler makes it easy to play with the various clustering options.  The basic faceted query that we've been doing is:

http://localhost:8983/solr/clustering?&q=*:*&fq=item_number_s:%22Item+%23%3A+185276%22

The primary source of data is the title, followed by the product bullets.  If we disable the carrot parameter for summarizing the product bullets, we get more clusters:

http://localhost:8983/solr/clustering?&q=*:*&fq=item_number_s:%22Item+%23%3A+185276%22&carrot.produceSummary=false including clusters for "Brick Paver" or "Glass and Clay Tiles".   It's great that there are more clusters, but you can see they become less interesting.

Another way of filtering down the volume of suggested clusters is to only cluster on the title field by blanking out the snippets field:

http://localhost:8983/solr/clustering?&q=*:*&fq=item_number_s:%22Item+%23%3A+185276%22&carrot.snippet= removes the clusters based on the product bullets like Help Contritube to LEED Certification of Projects.

Another way of changing what you get is to not try and filter what you cluster on by item number.  When you do a basic cluster on the full data set you get some great clusters on color:

http://localhost:8983/solr/clustering?&q=*:*

Lastly, you can play with different clustering engines, Solr comes with three of them, Lingo (the default), STC, and a Kmeans (http://wiki.apache.org/solr/ClusteringComponent#carrot.algorithm) one.  Try both the full data set and faceted data set to see the different types of results from the clustering algorthems.

STC seems to be very similar to Lingo, but does provide some clusters that have multiple labels.  For example MAPEI 10 Lb and Premium Unsanded Grout, as well as a seperate cluster of just MAPEI 10 lbs.
http://localhost:8983/solr/clustering?&q=*:*&fq=item_number_s:%22Item+%23%3A+185276%22&clustering.engine=stc
http://localhost:8983/solr/clustering?&q=*:*&clustering.engine=stc

KMeans is all over the places.  Lots of labels, but I don't quite see the connection between the items that makes them all clustered.  We have some clusters that are Tan, Chocolate, Cocoa, and three items associated with it, each one of the colors!
http://localhost:8983/solr/clustering?&q=*:*&fq=item_number_s:%22Item+%23%3A+185276%22&clustering.engine=kmeans
http://localhost:8983/solr/clustering?&q=*:*&clustering.engine=kmeans


What does it mean?

Well, first and foremost, it means that clustering can be a discovery tool for figuring out potential tags for your documents.  In this specific case, it really drove home that instead of 30 products with different sizes, colors, and types, that there are two products from a users perspective: MAPEI Unsanded Powdered Grout and MAPEI Keracolor U Unsanded Grout.   These two products should have a set of dropdowns for size and color.  The simple bit of clustering we did suggested that size was something that could be extracted as potential tags.   

You could model the products in Solr either as two documents, 1 for each product, and use dynamic fields to create multivalued fields for size and color.  Or, if you need to index everything denormalized the way Lowes did it, if you have identified some tags, then you can use field collapsing so that even though you may have 14 products all with the same name, but different sizes and colors you collapse on the item number, and then provide those data as distingushing drop downs.  

Here is a sense of what field collapsing on item number will do, we have 17 documents.   16 are unique item numbers, plus the 17th is the collapsed version of the 14 products that all share item number 185276.

http://localhost:8983/solr/browse?&q=*%3A*&wt=xml&group=true&group.field=item_number_s&group.main=true&rows=30


Oh, and just so you don't think this is an easy problem to solve, Amazon, who normally is a wonderful example of search done right, has the same exact problem for Aqua Mix Grout Colorant
 http://www.amazon.com/s/ref=sr_nr_scat_228926_ln?rh=n%3A228926%2Ck%3Agrout&keywords=grout&ie=UTF8&qid=1328909477&scn=228926&h=9d45480749457aa3cdff82998beda07f3a6316b7#/ref=nb_sb_noss?url=node%3D228926&field-keywords=aqua+mix+grout+colorant&rh=n%3A228013%2Cn%3A%21468240%2Cn%3A511228%2Cn%3A13397651%2Cn%3A228926%2Ck%3Aaqua+mix+grout+colorant






 
