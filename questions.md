# GameHub — Understanding the system

10 questions to test your understanding of the data flow and architecture.
Work through them in order: read the code first, then run the app, then try to break things.

---

## How to investigate

You will need three things:

**1. Read the source code**
Start with `models.py` (the schema), then `seed.py` (the data), then `app.py` (the logic).
Many questions are answered entirely by reading carefully.

**2. Run the app and interact with it**
Use the UI at `http://localhost:5000` or send requests with curl or Postman.
Observe what actually happens — don't just reason about it.

```bash
# Example: log an activity for nova (id=1) on Hollow Knight (id=1)
curl -X POST http://localhost:5000/activities \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "game_id": 1, "action": "started"}'
```

**3. Query the database directly**
Open `gamehub.db` with a SQLite tool and inspect the actual rows.

```bash
sqlite3 gamehub.db
.tables
SELECT COUNT(*) FROM notifications;
SELECT * FROM notifications WHERE user_id = 1;
```

Or use a GUI: **DB Browser for SQLite** (free, recommended).

---

## Suggested approach

| Phase | Questions | What you are doing |
|-------|-----------|-------------------|
| Read first | 1, 4, 8, 10 | Understand the code before touching anything |
| Then run it | 3, 6, 9    | Observe actual behaviour                     |
| Then break it | 2, 5, 7  | Try things, hit walls, reason about why      |

---

## Questions

**1.** When a user logs a new activity, how many database tables are written to?
List them and explain why each one is affected.

It's written to 2 tables. activities and notification. The notification is triggered automatically 
when create_activity() is called so both 2 datamobase are written in the same connection.

---

**2.** You call `DELETE FROM users WHERE id = 3` directly in SQLite.
What happens, and why? What would you need to do instead?

It will find the user_id with 3 and delete it. However, there is one thing to remark.
The command will delete the user row with the id = 3 BUT it may leave orphaned rows in activities, notification and etc.


---

**3.** User `nova` changes her username to `nova_2`.
She then checks her friends' notification feeds.
What do they see — the old name or the new one? Why?

Because the username is stored as a String value in the notification, notification will keep showing
the old user name which is "nova" not "nova_2." However, the new user name will show on the user_id field.


---

**4.** Trace the full journey of a `POST /activities` request.
Starting from the HTTP call, list every operation that happens before the response is returned.

Flask routes the request -> get_db(), datamobase connection -> INSERT blah blah -> conn.commit(), write doen
-> Runs Queries -> conn.commit(), second save -> conn.close(), close connection -> Return 201


---

**5.** `pixel_queen` opts out of activity tracking.
A teammate adds an `opted_out` boolean column to the `users` table and updates the `POST /activities` API route to check it.
Is the feature fully implemented? What did they miss?

Not really. The API route is there, but the HTML view route is not.
For example POST/activities. Our queen will still get the notifcation.
Also, there on no UI to toggle the opt_out settings

---

**6.** How many rows are created in the database when `nova` logs one activity, given the current seed data?
Show your working.

It is 4 rows total.
1 + 3 of nova's friends

---

**7.** You need to delete `maya_r`.
In what order must you delete rows across the tables, and why does the order matter?

We can follow delete_user() at app.py

1. DELETE FROM notifications WHERE user_id = maya_id — notifications maya received (they reference activities, so must go before activities)
2. DELETE FROM notifications WHERE triggered_by = maya_id — notifications her activities triggered for friends
3. DELETE FROM activities WHERE user_id = maya_id — her activities (now safe; notifications pointing to them are gone)
4. DELETE FROM user_games WHERE user_id = maya_id — her game library
5. DELETE FROM friends WHERE user_id = maya_id OR friend_id = maya_id — both sides of her friendships
6. DELETE FROM users WHERE id = maya_id — finally the user row

---

**8.** The `notifications` table has a foreign key pointing to `activities`.
What happens if you try to delete an activity that has notifications attached to it?

The deletion will be rejected with a FK constraint violation.
We have to delte the notification rows first, then the activity.

---

**9.** A bug is found in the game catalog — wrong genre for one game.
You fix it and restart the app to ship the change.
What else just went down, and for how long?

The entire application went down. Fixing one field in the games table requires restarting the whole Flask process.
In other words, every user is affected.

---

**10.** A teammate says: *"let's just move the notification logic into its own function in `app.py`"*.
Does that solve the problem described in Task 4?
What is the actual architectural issue?

No, it won't. It still remains the same, for example response will still be blocked until all notification inserts are completed.
So we should use microservices. MMMMM you said it the best. We have to do decoupling to make our service more stable