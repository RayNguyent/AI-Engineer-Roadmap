URLs:

https://abc.com/tim?video=123

https://abd.net/courses/python?utm_source=youtube &page=2

Domain               Path/Endpoint                    Query Params

# Request/Response

Request Components:

- Type/Method:
- Path
- Body
- Headers: security

Response Comp:

- Status Code: 404: not found, 200: success
- Body
- Headers

<aside>
💡

Us users find a website (aka client aka frontend), website sends request to API server, API server sends back response 

</aside>

Ex:

/books: API

GET:  /books : lists all books in the db

DELETE: /books/{bookID} : deletes a book based on their id

POST: /books : Creates a Book item

PUT: /books/{bookID} : update a book

GET: /books/{bookID} : retrieve

ex request/respone:

request comps:

- type: PATCH
- path: /api/post/45535
- Body:

```python
{
"title":"updated title",
 "description": "I dont like this caption"
}
```

- Headers:

```python
{
"Content-Type":"application/json",
"Authorization": "bearer sdifhsdjkhcsdo8"
}
```

response comps:

- Status code: 204
- Body

```python
{
"title":"updated title",
"description":"i dont like this caption",
"postID":"45535",
"updateAt":"Sept 26, 2025",
"createdBy":"user-123",
}
```

- Headers:

```python
{
"Content-Type":"application/json",
}
```

<aside>
💡

“Inject” = FastAPI automatically gives your function the value instead of you passing it manually

</aside>

<aside>
💡

Event Loop: runs continuously in a loopManages and executes tasks (coroutines)Switches between tasks when they are waiting (I/O)

- Runs continuously in a loop
- Manages and executes tasks (coroutines)
- Switches between tasks when they are waiting (I/O)
</aside>

> IO mean your program is **talking to something outside itself**.
ex: Calling an API (`requests`, HTTP), Querying DB (`SELECT * FROM users`),Reading/writing files,Sleep, external service response
> 

<aside>
💡

Keyword arguments = passing values using parameter names

ex: greet(name="Hoang", age=25), #order doesnt matter

ex: async def read_items(*, item_id: int = Path(..., title="The ID of the item to get", ge=1), 

q: str):    # *: forces everything after to be keyword arguments

</aside>

> @app.get("/items/{item_id}"):  take item_is from the URL
⇒ Handles GET request like: /items/2?q=phone
>