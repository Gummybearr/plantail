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

The first feature of PlanTail is creating a simple todo list. System generates a "task", a unit of todo list, when task name, start time and end time is given as input.

When a user creates a task, it is stored in MySQL(RDS). Considering the unexpectency of business development and ability to cope with diverse diverse access patterns, RDBMS was chosen as the main database.   

reference: https://www.datastax.com/ko/blog/relational-vs-nosql-when-should-I-use-one-over-the-other

2. create repeating task

Unlike non-repeating tasks, repeating tasks must be presented to the user on multiple occasions without creating redundant data entries. To achieve this, the "Task" entity's schema is modified.

The "repeat unit" field is an enumerated type that can be set to "Day," "Week," or "Month." The "repeat_schedule" field indicates how often the task is repeated within the specified repeat unit. For instance, if the repeat unit is "Week," the repeat schedule can range from 1000000, which means the task is repeated every Sunday, to 1111111, which means the task is repeated daily. An interesting decision made for the "Task" schema was to avoid using bitwise operations for the "repeat_schedule" field to enhance column readability. While 32 bytes consume more storage space than 4 bytes, the representation 0000111 is undoubtedly easier to comprehend than the single number 7.

```sql
CREATE TABLE `task` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint NOT NULL,
  `name` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `start_at` int NOT NULL,
  `end_at` int NOT NULL,
  `max_streak` int NOT NULL,
  `current_streak` int NOT NULL,
  `repeat_unit` varchar(8) NOT NULL,
  `repeat_count` int NOT NULL,
  `created_at` bigint NOT NULL,
  `updated_at` bigint NOT NULL,
  `repeat_schedule` varchar(32) NOT NULL DEFAULT '1',
  PRIMARY KEY (`id`),
  KEY `ix_userid_startat_endat_repeatunit` (`user_id`,`start_at`,`end_at`,`repeat_unit`),
  KEY `ix_userid_startat_repeatunit` (`user_id`,`start_at`,`repeat_unit`),
  KEY `ix_userid_endat_repeatunit` (`user_id`,`end_at`,`repeat_unit`)
) ENGINE=InnoDB AUTO_INCREMENT=497 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;;
```

3. query tasks of day

Querying tasks of a specific day is straightforward using the pseudo-code. 

```go 
func QueryTask(request QueryTaskRequest) QueryTaskResponse {
	// query user
	var user domain.User  = query user by id

	// query unRepeating tasks
	var unRepeatingTasks []domain.Task
	mySqlConnection.Table(infra.TaskTable).Where("user_id = ?", request.UserId).Where("start_at = ?", request.Date).Where("repeat_unit = ?", domain.NEVER).Find(&unRepeatingTasks)

	// query repeating tasks
	var repeatingTasks []domain.Task
	mySqlConnection.Table(infra.TaskTable).Where("user_id = ?", request.UserId).Where("start_at <= ?", request.Date).Where("end_at >= ?", request.Date).Where("repeat_unit != ?", domain.NEVER).Find(&repeatingTasks)
	repeatingTasks = lo.Filter(repeatingTasks, func(task domain.Task, _ int) bool {
		return task.IsIncludedInDatetime(request.Date)
	})

	// merge unRepeatingTasks and repeatingTasks
	var totalTasks []domain.Task
	totalTasks = append(totalTasks, unRepeatingTasks...)
	totalTasks = append(totalTasks, repeatingTasks...)

    // map tasks to dto and then return
    return ...
}
```
4. update task

In this app, making changes to an existing task is straightforward, assuming you've already created one. You can edit the task's name and choose whether it repeats regularly. If it does, you can even set its frequency and when it should be shown to you.

5. delete task

While deleting tasks, PlanTail currently employs the simpler "hard delete" method. This means tasks are permanently removed from the system. However, the preferred approach is "soft delete," where tasks are hidden instead of erased. This offers protection against accidental deletion and allows for easier recovery in case of mistakes. Though not currently implemented, soft delete is likely a more robust solution for future versions of PlanTail.

reference: https://docs.sourcegraph.com/dev/background-information/postgresql#:~:text=Hard%20deletes%20are%20hard%20to,once%20you%20determine%20what%20happened.

# v1.1.1
1. success task

In PlanTail, marking a task as complete takes center stage. To achieve this, the app creates a new entry in the "log table." This simple yet effective approach allows the system to effortlessly determine if a task is done by simply checking if a log entry exists for that specific task and date.

Schema of the log table is as follows

```sql
CREATE TABLE `task_complete_history` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint NOT NULL,
  `task_id` bigint NOT NULL,
  `completed_at` bigint NOT NULL,
  `score` int NOT NULL,
  `created_at` bigint NOT NULL,
  PRIMARY KEY (`id`),
  KEY `ix_userid_completedat` (`user_id`,`completed_at`) USING BTREE,
  KEY `ix_completedat_userid` (`completed_at`,`user_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1164 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;;
```


While traditional databases enforce data consistency through "foreign key" constraints, PlanTail takes a different approach. Instead of relying on the database, the app itself verifies data integrity by implementing a "logical foreign key" system. This means the application code actively checks if referenced entities exist, ensuring data consistency without introducing performance issues or potential deadlocks that might arise from foreign key constraints. While foreign keys are valuable for domains requiring high data consistency, PlanTail's chosen method offers a balance between performance and data integrity.

reference: https://softwareengineering.stackexchange.com/questions/375704/why-should-i-use-foreign-keys-in-database

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

# v1.6.1
1. make users select day | day of month when creating repetitive tasks.
```
```

# v1.7.0
1. add push for tasks
```
```

# v1.8.0
1. add a reward that is triggered when a user completes all tasks of the day.
```
```

# v1.9.0
1. open weekly task calendar
```
```

# v1.9.1
1. tab any day in week to open daily calendar
```
```

# v1.10.0
1. generate daily task image
```
```
2. share generated daily task image
```
```

# v1.11.0
1. open social tab
```
```
2. add leaderboard in social tab
```
```
3. add user info query modal in social tab
```
```
4. add friend task page in social tab
```
```