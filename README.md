# PennBook

## Instructions

Update AWS credentials in `database.js`. Run `npm install` to install dependencies. Run `npm start` to start the server. Navigate to `localhost:3000` to view the application.

## Description

Our system is a more bare-bones version of Facebook. In addition to the required features that were laid out in the write up, we have also implemented a number of extra credit features, including but not limited to sending images in chats, infinite scroll in news search, and various privacy controls. We have utilized the stack that was detailed in the writeup, in addition to writing the application with Node.js and React.js. Out Frontend was written with JavaScript, CSS, Bootstrap, and MUI. The “smrt” in smrtBook comes from our four first initials!

## Components

### Login/Register

We expanded on the Login/Register Pages that we built in our homeworks to include the other features necessary for our mini-Facebook, including news interests and affiliation. We found that the best way to enable users to select multiple news interests was through a list of checkboxes. We also incorporated input validation to check if the user put in the right values for each input field and also selected at least 2 news interests. Once the user has successfully logged in or signed up for an account, we automatically create a session instance for them in the backend to indicate that they are an active user. We store all of the initial information about the user in our “users” table in DynamoDB. The primary key for this table is the username, and we ensure that this is maintained by checking to make sure that the username is not taken when a user is trying to sign up for an account. We decided that the primary key should be the username for ease of search, since we knew that we would be frequently looking up information about a specific user. Note that all passwords are hashed in the backend with SHA3.

### Account Settings

On render, we load the user’s existing information into the fields of our account settings form to remind the user what is already there in the case that they do not want to change some fields and only want to change others. Similarly to Login/Register, we have robust input validation as well as password validation comparing hashed passwords only. We ensure that the user knows their preexisting password before we allow them to change to their new password for privacy and security.

Additionally, there exist options for the user to change the privacy settings in the account settings page, which allows the user to set their wall page visibility to public, friends-only, or private. These changes take place immediately and the attribute of visibility is unique and accessible only to each user

### Home/Wall Pages

Upon visiting a wall page for another user, represented by the URL /user/{friend}, we can get all the posts that are associated with a user - namely, the posts that the user has created on other peoples’ walls, and other posts that friends have posted on their wall.

A sort operation is performed on all posts associated with a wall or home page based on timestamp, which is generated at the time of post creation, to guarantee consistency when rendering and display.

The home page follows a similar operation to generate relevant posts, though it has an additional step in the beginning - which is to query the current user’s friends, push them all into an array, and gather all of the postIDs associated with individual users in a similar fashion as described above.

### Posts and Comments

Designing the DynamoDB schema to represent all of our posts was a significant challenge in displaying the home and wall pages. We prioritized efficient and consistent operations when designing these tables. We have two tables, posts and posts_new, with duplicate information that both have a primary key as postID (which is unique, generated with uuid tags), and a secondary key as creator and wallUsername, respectively. Whenever a username is identified as someone’s wall, it queries both tables by the secondary key (using a GSI/Global Secondary Index) and gathers all associated postID from the DynamoDB tables. However, since duplicate information can be retrieved (as a user can post on their own wall, resulting in duplicate IDs from these two operations), we resolved this error by feeding in all postIDs into a set and querying on each of the elements in the set and gathering the required information thereafter.

Our comments table takes in a primary key commentID (as all comments have to be unique) and a secondary key postID. This made the most sense to us as a secondary key as postID allows us to query with a GSI (Global Secondary Index) to gather all comments associated with a postID and display them sorted by timestamp for each post.

### User Search

We created a special search table to generate suggestions to allow for dynamic search. An entry is added to this table when a user creates a new account in order to maintain consistency. We use a query with a key condition expression “begins with” to find up to 15 entries that match (prefix-wise) with the entered search term. Each result also has the ability to request a friend, and all friend requests are persistent and dynamically updated.

### Friends List

We render the list of friends of each user by simply looking up our user in our users table. We can also see the pending friend requests that the user has in this way. We created a few buttons for accepting, rejecting, or removing a friend. This required a dual-sided acceptance/rejection/removal in order to maintain consistency due to the nature of our users table. We dynamically rendered this page so that any changes in other tabs would be efficiently reported back to the user, and handled state changes on the frontend before the backend finished so that the user would have the lowest latency experience.

### Friends Visualizer

We used the ReactFlow package to build our friend visualizer. ReactFlow works very similarly to the package detailed in the example code. We found the friends of friends that share the same affiliation as the user by first finding all of the friend’s friends, and then making consecutive calls to check each friend’s affiliation. This was the most efficient method given our table structure since we did not make the primary key or the sort key affiliation. Creating a table with this primary key would not be a wise decision because we rarely need to search by affiliation outside of this instance. Thus, instead of scanning the table to find users who are a friend of a friend and share an affiliation with the original user, we just ran multiple queries.

### Dynamic Content

Content that needs to be dynamically rendered is stored as an internal state in React so that any new additions to the page (new comments, new posts) will be visible immediately on the page without flickering. Additionally, a performUpdates() function is called every 10 seconds to check for any differences in the posts and comments when fetched and to replace any differences in the posts and comments synchronously across windows.

### Chat

Our chat relies on socket.io to instantly receive messages from other people connected to the same server. For maintaining persistence, we decided to create a table for chats where each chat has an attribute containing the messages that were sent in that chat; we also have another messages table which contains data about each message. In the users table, we stored a reference to the chats that each user was a part of—each chat had a unique identifier which contained all of the usernames (in lexicographical order) and the # instance of that chat that currently exists (i.e. [username1]_[username2]_...\*[username7][3] for the third such chat with those 7 people). We chose to design it this way so that we can easily get from the chat’s id to the users in the chat and from the users to all of their relevant data. In addition, we intentionally chose to minimize the amount of data in the attributes of any given table (we decided against storing all of the messages as an attribute within the chats table).

Since chat was very dynamic in that users can constanly enter and leave chats (while sending messages in between), it was crucial that all of these table are in alignment with each other. This required carefully tested database calls, making sure that they accurately reflected what happened in the chat. While on the chat page, we pulled the current user from session cookie and then we emitted sent messages to the specific group chat that it was sent to on socket.io. Users automatically join the rooms (via socket.io) corresponding to the chats that they are a part of.

### Adsorption Algorithm

Our data analysis backend runs through a Java Spark program as per the specifications. We ultimately decided it was best to run this completely independently of the frontend, and simply store the results into the database for the actual app to then simply display. This was decided because the job takes a significant amount of time (especially with a lot of users) and just needs the data from the frontend, not any other input. This resembles how major platforms run their data analysis occasionally (for example Google with its pagerank) and does not allow this to impact the performance of its service. We did not implement this job on-demand - the most important factor in this rather than not being able to set it up was that in a real application, many users would simultaneously change interests, and it was highly unclear how we would then go about synchronizing these multiple calls since they would update the same table. We also felt this was alright as the user would still see new suggestions within the hour from the timed job.

Overall, this design was effective in maintaining the backend’s independence, and not allowing this kind of work to affect the user experience in any way when using the other features provided.

### News Search

Using the notion of the inverted table, search results from the entire article database are sorted and shown to the user on this page. This page implements infinite scroll, which we felt was fitting since queries could return a massive number of results that would exceed the throughput of DynamoDB if done all at once. The “liking” functionality from the newsfeed is also present here.

### News Feed

The results of the adsorption algorithm are loaded into the DynamoDB “users” table, where suggested article IDs are stored. We dynamically render these in the News Feed, so that whenever the Adsorption Algorithm finishes running, the DynamoDB data changes, and the frontend changes as well. We give users the ability to “like” different news articles, which is also stored in our “users” table, and then fed back into the adsorption algorithm.

## Changes and Lessons

Storing Arrays of Comments and Chat Messages in DynamoDB
When we were initially designing our posts database, we added an attribute of a list of comments representing all comments that were linked to a post. We similarly created a list of chat messages for each chat in the chat database. However, we realized that this would compromise the scalability of the system due to the limited size of DynamoDB’s support for arrays, and decided to refactor our code to support comments and chat messages in a separate table, unbounded by size. Note that we still store friends, likes, requests, etc. in arrays. This is because strings are lightweight data structures and we can store tens of thousands of strings before the array runs out of space.

### Adsorption Algorithm

More so than in the third homework, the adsorption algorithm requires careful consideration of efficiency, simply because much more data is being propagated through the network than in simple pagerank. It was initially very unclear how to store so much data. I considered using different RDDs for different users, but realized this would be inefficient both in terms of code and performance. After reading through the Youtube paper, which used vectors to store each label weight on each node, I realized that Java maps went along with the requirements well, and ultimately came to propagate around mappings of users to nodes. This implementation was effective as it allowed for the use of map library operations like merging to aggregate together data throughout the algorithm, leading to clean code that performed no unnecessary work.

Another complication was choosing appropriate cutoff values to stop propagating labels and to stop iterating all together. Both of these we ultimately determined through trial and error. Due to the fact that the news suggestions had to be from the current day and there were only a small amount of articles each day, we had to choose sufficiently small cutoffs such that the labels could actually get to current news articles. This cutoff ended up being quite small just to recognize these node, and was simply determined through trial and error. Similar reasoning led to the cutoff to stop iterating. Using print statements in the livy job, I determined the approximate point at which the ordering of articles was constant and later iterations were simply unnecessary, and just stopped the job there. This cutoff was significantly larger than the propagate cutoff because this relative ordering became constant significantly quicker in our test cases.

We implemented the following additional features:

- LinkedIn Style friend requests, with the ability to accept and deny
- Images hosted through Amazon S3, with the ability to send images through chat
- App wide chat notifications
- Add to existing chat
- Server-side sessions for increased security
- The ability to delete your own comments and posts, but not others’
- Privacy Levels: Public, Friends Only, and Private privacy settings, can be changed in Account Settings
- Infinite Scroll in News Search
- Anonymous Commenting
- Groups: group search, joining groups, creating groups, group pages
