---
full_name: Social Media Support Application
short_name: SMSA
image: assets/images/SMSAMaindarkgrey600x600.png
shortDesc: An application designed to save time by automating posting to Social Media.


---
# Social Media Support Application

The Social Media Support application is the largest application I have worked on. It is designed to allow users to more easily share large amounts of already existing data with the public via Twitter. 

The application takes data from existing excel files, translates the contents into human readable texts and stores it on a MongoDB database. Then when the application is run, a scheduler runs in the background which knows when a post should be sent to Twitter. A front end GUI was created by my team which allows users to edit information about the post including when a file will be posted, any any other information.

There are five main facets to the program which I will cover in order, with code samples when necessary for illustration or as a sample of my work. The first is the Post Builder.

##  [Post Builder](#post-builder) 

The post builder is the starting point of the system. This is where the excel file that was provided to us is extracted and it’s also the system that is activated when someone wants to update the database with a new excel file.

How we did this was to loop through each row and column gathering the data from the nine columns.

 For example on the first two lines you can see the workbook being opened, a worksheet being assigned and then iterating over the work book.

 On line 13 we assign an elementCode, and on line 17 we assign speciesType. If we were to go through the rest of the code we'd see something similar (but distinct) for how each column is handled due to their own unique data transformation requirements.


{% highlight python linenos %}
workbook = xlrd.open_workbook(xlsx, encoding_override='cp1252')
worksheet = workbook.sheet_by_index(0)
Order = []
random.seed()
for row in range(1, worksheet.nrows):
imageLink = None
scientificName = None
elementCodeExists = True
photoExists = False

for col in range(9):
if col == 0:
elementCode = '' + unicodedata.normalize("NFKD", worksheet.cell(row, col).value)
if elementCode == '':
elementCodeExists = False
elif col == 3:
speciesType = '' + unicodedata.normalize("NFKD", worksheet.cell(row, col).value)

{% endhighlight %}

Once the translation process is finished we need to take all the data we've collected in variables and assign it into a dictonary. This is the format we're going to send the information to MongoDB in.

{% highlight python linenos %}
postDict = {
                "_id": int(row),
                "Name": fullName + "\n",
                "Species Type": speciesType + "\n",
                "BC List": bcList + "\n",
                "COSEWIC Status": Cosewic + "\n",
                "Provincial Status": provincialStatus + "\n",
                "User Stories": "" + "\n",
                "More Info": moreInfo + "\n",
                "Image Link": imageLink + "\n",
                "Hashtags": "" + "\n",
                "Element Code": elementCode
            }
if testMode == False:
                x = postsCollection.insert_one(postDict)
{% endhighlight %}

After that, all that's left for Post Builder is to determine the current time to assign a date for this post to be published (which can be changed later, we found it useful for posts to have default dates) and then repeast the process.


{% highlight python linenos %}
        dateNow = datetime.datetime.now()
        dateNow = dateNow + datetime.timedelta(days=1)
        firstPostStr = str(dateNow.year) + '-' + str(dateNow.month) + '-' + str(dateNow.day) + ' ' + str(dateNow.hour) + ':' + str(dateNow.minute) + ':' + str(dateNow.second)
        firstPost = datetime.datetime.strptime(firstPostStr, '%Y-%m-%d %H:%M:%S')
        firstIteration = True
        i = 1
        for post in Order:
            if firstIteration:
                postDate = firstPost
             
                firstIteration = False
                postOrderDict = { "PostOrder": int(i), "PostNumber": str(post), "PostDate": str(postDate)}
                if testMode == False:
                    x = postOrderCollection.insert_one(postOrderDict)
                
                i += 1
            else:
                postDate = postDate + datetime.timedelta(days=7)
                
                postOrderDict = { "PostOrder": int(i), "PostNumber": str(post), "PostDate": str(postDate)}
                if testMode == False:
                    x = postOrderCollection.insert_one(postOrderDict)
                
                i += 1
{% endhighlight %}

All of this was behind the scenes, all this event gives us is a nicely populated MongoDB database to pull from later on, which the Graphical User Interface will display to our users.

![MongoDB Post Build](/assets/images/MongoDBPostBuild.PNG){:class="img-responsive"}


## [File Manager](#file-manager)

Test.