# v1.0.0
1. signin

Three-step user sign-in process
1. Validate if user exists. This is done by using the unique constraint on the user's email address.
2. Validate if the hashed user secret is consistent with the existing one. Although we don't save the user's plain password to our database, we can still check if the user typed the correct password because we have the user's unique salt stored in our database.
3. If the user is valid, generate a JWT token as the response. We use token-based authentication for overall system simplicity. 

reference: https://www.youtube.com/watch?v=zt8Cocdy15c


2. signup(email, pw, nickname)

Signing up with email and password is one of the most widely used method of creating a user. Since password is a private and personal data that should be secretly handled and encrypted in a way that user given plain text should not be retrived, we store encrypted hash instead. The password is encrypted in one-way, bcrypt algorithm to be specific, with a salt that is unique for each users. This way, users can be assured that they are the only ones who can infer the password since modern computers won't be able to guess it in their lifetime. 


# v1.0.1
1. auto signin

Client stores two values in its local storage.
First, client locally keeps whether if user wants to use auto signin feature. This is a boolean value referenced when use opens the application. Second, JWT token user received when user signed in is stored. If user don't want to automatically login, this value is erased when user closes the application. 

One thing to notice here is that we only use access token that is available for a month. Since application only allows verified devices, refresh token is not necessary and we want users to sign in again if their token is a month old.


# v1.1.0
1. create non-repeating task

The first feature of PlanTail is creating a simple todo list. System generates a "task", a unit of todo list, when task name, start time and end time is given as input

2. create repeating task
```
```
3. query tasks of day
```
```
4. update task
```
```
5. delete task
```
```

# v1.1.1
1. success task
```
```
2. calculate score
```
```

# v1.2.0
1. view my info
```
```

# v1.2.1
1. toggle to show score or rank in my info
```
```

# v1.3.0
1. summarize my day using generative ai
```
```

# v1.3.1
1. support 3x2 widget for Android
```
```

# v1.4.0
1. use can crud diary on specific day
```
```

# v1.5.0
1. use social login(Google)
```
```

# v1.6.0
1. share link to other users
```
```
