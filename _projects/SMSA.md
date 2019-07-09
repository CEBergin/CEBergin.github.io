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

## <a name="post-builder">[Post Builder](#post-builder)</a>

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

Once the translation process is finished we need to take all the data we've collected in variables and assign it into a dictionary. This is the format we're going to send the information to MongoDB in.

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

After that, all that's left for Post Builder is to determine the current time to assign a date for this post to be published (which can be changed later, we found it useful for posts to have default dates) and then repeats the process.

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

## <a name="file-manager">[File Manager](#file-manager)</a>

The File Manager is the glue that holds our system together. It's in charge of the all the communication between the database and the rest of the system. Any way you want your data sliced, this is the subsystem that is responsible for that. For example, above we talked about how we sent a dictionary to the database with all the information about a post. Well if we want it back it's as simple as running dbReadFile as shown below:

{% highlight python linenos %}
def dbReadFile(self, fileName):
myquery = { "_id": fileName }
docDictonary = dict()
mydoc = self.postsCollection.find(myquery)
for x in mydoc:
docDictonary = x
return docDictonary

{% endhighlight %}

What if we wanted it as a string? Well we got you covered there too.


{% highlight python linenos %}
def dbGetFileString(self, fileName):
fileDictonary = dict()
fileDictonary = self.dbReadFile(fileName)
fileString = (
"Name: " + fileDictonary['Name'] + # 0
"Species Type: " + fileDictonary['Species Type'] + # 1
"BC List: " + fileDictonary['BC List'] + # 2
"COSEWIC Status: " + fileDictonary['COSEWIC Status'] + # 3
"Provincial Status: " + fileDictonary['Provincial Status'] + # 4
"User Stories: " + fileDictonary['User Stories'].strip() + " " + fileDictonary['Hashtags'] + '\n' + # 5
"More Info: " + fileDictonary['More Info'] + # 6
"Image Link: " + fileDictonary['Image Link'] # 7

)
return fileString

{% endhighlight %}

Want to update just a single database row?

{% highlight python linenos %}
def updateDBRow(self, id, dictOfValues):
myquery = { "_id": id }
newvalues = { "$set": { 
"Name": dictOfValues['Name'] ,
"Species Type": dictOfValues['Species Type'],
"BC List": dictOfValues['BC List'],
"COSEWIC Status": dictOfValues['COSEWIC Status'],
"Provincial Status": dictOfValues['Provincial Status'],
"User Stories": dictOfValues['User Stories'],
"More Info":dictOfValues['More Info'],
"Image Link": dictOfValues['Image Link'],
"Hashtags": dictOfValues['Hashtags'],
"Element Code": dictOfValues["Element Code"]
} }

self.postsCollection.update_one(myquery, newvalues)
{% endhighlight %}

What if you want just one specific row?
{% highlight python linenos %}
#GETS ONE SPECIFIC VALUE
def getValue(self, fileName, value):
fileContents = dict()
fileContents = self.dbReadFile(fileName)
return fileContents[value]
{% endhighlight %}

Okay, granted most of these are pretty specific queries, but they're all necessary for the system to function. There are a bunch of other ones, but I think you get the gist.

Something more interesting might be our shiftToPosition function which is what allows users to change a Posts' Post Order from the GUI (we're getting to it) and it's slightly more logically complicated.

{% highlight python linenos %}
def shiftToPosition(self, id, newPosition):
myquery = { "PostNumber": id }
myDoc = self.postOrderCollection.find_one(myquery)
currentPosition = myDoc['PostOrder']
myquery = { "PostNumber": id }
newvalues = { "$set": { 
"PostOrder": 0
} }
self.postOrderCollection.update_one(myquery, newvalues)

if currentPosition < newPosition:
count = currentPosition 
while count < newPosition:
myquery = { "PostOrder": count + 1 }
newvalues = { "$set": { 
"PostOrder": count
} }
self.postOrderCollection.update_one(myquery, newvalues)
count = count + 1
myquery = { "PostNumber": id }
newvalues = { "$set": { 
"PostOrder": newPosition
} }
self.postOrderCollection.update_one(myquery, newvalues)
#if we are below the new position:
elif currentPosition > newPosition:
count = currentPosition 
while count > newPosition:
myquery = { "PostOrder": count - 1 }
newvalues = { "$set": { 
"PostOrder": count
} }
self.postOrderCollection.update_one(myquery, newvalues)
count = count - 1
myquery = { "PostNumber": id }
newvalues = { "$set": { 
"PostOrder": newPosition
} }
self.postOrderCollection.update_one(myquery, newvalues)
else:
#print("no changes")
myquery = { "PostNumber": id }
newvalues = { "$set": { 
"PostOrder": newPosition
} }
self.postOrderCollection.update_one(myquery, newvalues)
{% endhighlight %}

Lines 1-7 work in relatively straight forward way. We find which document we're talking about based on the id that was passed into the function, then we find the current Position of the Document. From there we're updating the current posts position to 0 in the queue to keep it out of the way while we move every other position around.

At lines 10, 26, 40 we have the three possibilities when it comes to moving a post around in it's order. Either the posts new position is greater than where it was currently, it's less than where it is currently, or we just tried to move a post to the position where it already is. Obviously that last one is pretty easy to handle, we're just setting it back to where it already was, no need to change it.

For the other two however, some mild explanation.

On line 10 we note that if our current position is less than our new position we have to shuffle everyone up one value until there is a vacant position where we want to insert our document, then assign it the number we wanted.

On line 26 we essentially do the reverse. If the current position is greater than the new position we want to move to, we have to decrement everyone until there is a vacant position where we want to insert our document.

Looking back at this it's clear we could have used a function here for both cases with it either just being positive or negative, but that's the benefit of hindsight for you.

Anyways, those are some examples of how are file manager works as the glue that holds everything together. Next up we have the scheduler, the system that monitors time. 

## <a name="scheduler">[Scheduler](#scheduler)</a>

The Scheduler is our system for constantly monitoring time. The Scheduler essentially has to only be aware of three things: what time it is, what's the next post to be posted, and what time does the next post to be posted, post at.
