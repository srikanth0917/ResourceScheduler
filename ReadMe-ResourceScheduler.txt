
-->Developed Project as a standalone java application using Eclipse (Java 1.7), Junit 1.4.

-->Test classes are inside the source folder
-->Execution starts from MessageEngine Class (main method in it)

Explanation for the approach:

1) Created two Interfaces as per given requirement
    -Message
    -Gateway

2) Message is an interface (basically abstract type of message) Resource Scheduler will be getting specific types
   assumed JpMorganMessage is one of the types of messages it gets from the source

3) JpMorganMessage class is created which implements Message interface
   It will have fllowing fields 
   messageID---->Unique id for each and every message
   groupID------>represents the group of the message it belongs to ( it will help us to find send the particular group of messages
                 if there are any messages already being processed.)
   completed---->Indicates whether the message is completed or not
   groupCancelled----->It represents the specific group has been canclled (Canclled group messages wont be send to the Gateway)
   messages------> to store the messages.

   methods:

   createmessage method:
   As we dont have any source to get the messages from, so I have created this method to just create messages so that I can use the mesages
   for processing.Number of messages will be sent to this method as a parameter here I set is as '5'.
   
   removeCanclledGroupMessages:
    
   To remove the canclled group messages from the list, I have created this method. if we remove the cancelled group of messages from the list
   then these wont be sent to the gateway

   completed:
   
   JpMorganMessage implements Message interface we shoud implement completed method, this is used to set the completed indicator to ture
   whenever this method is called. This is called whenever the processing is completed for the message.

  remaining methods are getters and setters for the instance variables.


4) ResourceChecker class:

    purpose: Made it as a thread (implements Runnable), to keep track of the resources and the messages (what resource holds the what message). ResourceCheker checks which resource is available for processing it gets the resource id. 

    Instance variables:   

         resourceAvailabilityMap---->it maintains the available resources in this map
                                     key-->resourceID
                                     value--->true/false (false-->available, true--->not available(running))

          inProgressMessages----->it maintains the in progress messages associated with resources
                                  key-->resourceId
                                  value--->message

     Methods:
        
      addInProgressMessages:
      
      It adds the message and ResourceId as key, value pair into inProgressMessages ConcurrentHashMap. It will be usefull when checking for the messages in a resource which are in progress.
      
      getNextAvailableResource:
     
      it gets the next available resource from the resourceAvailabilityMap, when we receive a message we go and check for the resource availability.
      if the value for the resourceId (key) in the Map is false then it is available, if the value is true then the resource is not available.
      we loop through the keySet and gets the values for the each key if the value for any resource is false then we return the resourceId(available resource)

      isResourceAvailable:
      It checks for the available resource, if the resourceAvailabilityMap contains any 'false' then atleast one resource is availble to process the messages.
      
      run:
      this will get its chance after we start the thread, whenever the thread gets its turn run method will be called.
      here we process the removal of resourceId from the inProgressMessages if the associated message is completed its processing (if the completed indicatior is true)
      and once we remove the resourceId from the inProgress map the resource completed its task, it should be made available for the use. so we call updateAvailableResource for that task.


     updateAvailableResource:
     it updates the resourceAvailabilityMap with the arguments passed to this method, as mentined above intially we set up the resourceAvailabilityMap with the number of resources with fasle (value)
     once they start doing its job this method will be called and resourceId will be updated with "true" in the map (its running).
     if the resource completes its job then resourceId will be updated with false in the map (its available)

5) MessageChooserStrategy Interface
   -getNext
   -hasNext
   -processMsg
  
   this is to give a platform for the classes which implement this interface should have these messages. becuase these methods will be called in the logic.
   we can have multiple strategies (algorithms) to implement this, all the classes which implements this have to provide functionlaity for the above methods.
  
6) MessageChooser
   this will be used as a handle to call the methods in classes which provides the implementation for the MessageChooserStrategy.
   
   its constructor takes MessageChooserStrategy reference as a an argument, whatever the classes that implements this interface can be passed to this constructor.
   In turn created object will be used to call the implemented methods in their respecitve classes.

7) GroupPriorityStrategy 
   
   It implements the MessageChooserStrategy, it provides implementation for the above methods.
   actual proessing of messages (Listing the messages in the Queue for processing) based on the priority is implmented here.

   we actually pass this object as an argument to the MessageChooser, using the MessageChooser handle we call the methods in this class.
     
     Instance variables:
        prioristisedMessages--> its for the list of prioritised messages that need to be processed.
        processingMsgs--------> its for the list of processing message groups 
 
    methods:
      processMsg
         -checkForTheGroup
         -getTheFirstMsgFromGroup    
         -adjustPriorityList
      
     step 1: when we receive the first message it will be added to the prioritised list and the processing group messages list
     step 2: if it is not empty then we check for the processing group message list for the group id.
     step 3: if the group id is there that means the type of group is already running at the resources side, so we should retreive the first message of that group 
             in the prioritised messages list.
     step 4: If the same group message exist in the priotirised list then we should be adding the existing message to the start of the prioritised list.
     step 5: Add the recevied message to the end of the list.
     step 6: if there is no same group message in the priotirised list then just add the received message to the prioritised list.
     step 7: if the prioritised list is not empty and there is no existing group id in the prcoess messages list for the received message
              then we add message to the both the lists.
   
     checkForTheGroup:
      it checks for the group id in the processing messages list, if it finds one then it returns true else returns false.


     getTheFirstMsgFromGroup:
       it loops through the prioritised list of messages if it finds the same group id in the list and that message is not completed its processing then
       that will be returned. if not null will be returned.

     adjustPriorityList:
       It adjusts the list with the given message by adding it to the start of the list (which should be processed next)

   remaining methods are setters and getters

8) MessageEngineConfiguration class

	It configures the number of resources and the number of messages needed. Also it assigns the messagechooserStrategy as GroupPriorityStrategy
        In which we have the business logic to process the messages.
        if we implements a new algorithm ( in a new class) we can change the strategy by passing the newly created object to the change strategy method in
	MessageChooser class, we dont need to modify any other code.
	
9) MessageEngine 
   
   It implements Gateway interface.

    Instatnce variables:
	messageChooser---> this will be used as a handle in calling the processing messages logic.
        resourceChecker----> this acts a resource checker for each and every message we receive (its a thread)

    MessageEngine-- constructor
          its one argument constructor which takes MessageEngineCOnfiguration object.
          Instantiates the resourceChecker object and messageChooser object.

     receiveMessage
          --processMessage
          --updateAvailableResource
          --addInProgressMessages
          --send
          --complete
      
     step 1: if the resources are not available we dont send any messages to the Gateway, we just keep on
             processing them ( priortising messages will be done)
     step 2: if the resources are available we get the next resource available then start the resource checker thread
              
     step 3: Once the thread is started, update the resource as a message assigned to it (resourceAvailabilityMap will be updated)
     
     step 4: If the Priority messages list has got the messages then we retreive the message from the lsit.
     step 5: we add this resource id and the message to the inProgressMessages map.
     step 6: we call send and complete methods
     step 7: if the priority messages list is empty and the resource is available then we straight away send message to the Gateway
 
   main
   --------
     It creates the required objects for the MessageEngine to work properly
          --JpMorganMessage object is for creating messages and accessing the methods in it
          --MessageEngineConfiguration object is for retreiving the number of resources and messages.
            and also to pass it to the MessageEngine constructor.
          --MessageEngine constructor to invoke the MessageEngine methos.
         
     step 1: once we receive the message check for the cancelled group messages if there is one then we remove the 
             group related messages from the list. These canclled messages wont be send to the Gateway
     step 2: checks for the number of resources available if there are zero resources then displays the error message
      
     step 4: if there are same number of resources to the same number of messages then we direclty send the messages to the Gateway
      
     step 5: Else we call the receive message method by passing each message at a time.

_____________________________________________________________________________________________________________________________________________
_____________________________________________________________________________________________________________________________________________

Solution for the Production Quality would be,

1) By using Produer-consumer approach 


		


   
    




     
       
      



     