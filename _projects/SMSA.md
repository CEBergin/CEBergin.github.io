---
full_name: Social Media Support Application
short_name: SMSA
image: assets/images/SMSAMain.PNG
shortDesc: An application designed to save time by automating posting to Social Media.


---

The Social Media Support application is the largest application I have worked on. It is designed to allow users to more easily share large amounts of already existing data with the public via Twitter.

The application takes data from existing excel files, translates the contents into human readable texts and stores it on a MongoDB database. Then when the application is run, a scheduler runs in the background which knows when a post should be sent to Twitter. A front end GUI was created by my team which allows users to edit information about the post including when a file will be posted, any any other information.

Below is some sample code I wrote that is constantly monitoring the launch time of a given file:

{% highlight python linenos %}
    def checkTime(self):
        fm = FileManager.FileManager()
        while(True):
            
            time.sleep(10)
            currentDT = datetime.datetime.now()
            self.getLaunchTime()
            isPast = self.checkIfTimeIsPast(str(self.launchTime))
            

            #print("self.launch time is: " + str(self.launchTime))
            #print("is past is: " + str(isPast))
            if isPast == True:
                #print("time is past")
                fm.dbCreateNewTimes()
                self.getLaunchTime()

            if currentDT.day == self.launchTime.day and currentDT.hour == self.launchTime.hour and currentDT.minute == self.launchTime.minute:

                #print("COMPLETE, LAUNCH IS GO")
                #This gets sets the Schedule object to have the text of the top post text file in PostOrder file
                self.getPost()
                #send to twitter
                x = TwitterPrototype.TwitterConnect()
                x.extract_keys_from_db()
                x.authorize()
                x.update_api()
                #print(self.stringPayload)
                x.post_to_wall(self.stringPayload, self.toPostDictonary)
                #print("Posted: \n " + self.stringPayload)
                logEntry = self.stringPayload.replace('\n', ' ')
                logEntry = "[" + str(datetime.datetime.now()) + "] " + logEntry
                fm.writeLog(logEntry, "log.txt")

                #Here we need to call functions to move the current file to the bottom of the list, and get the next one
                
                fm.shiftToLastPosition()

                self.getLaunchTime()
                #BEFORE THIS CALL A WIPE FUNCTION
                self.gui.refresh()
{% endhighlight %}