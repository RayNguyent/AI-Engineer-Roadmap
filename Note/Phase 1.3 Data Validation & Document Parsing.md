*Pydantic*

Pydantoic can turn type hints into runtime validation rules.

Instead of writing dozens of if `isinstance()` checks and custom validation functions, you define your data structure **ONCE** using familiar Python syntax.

⇒ Pydantic handles the rest: validating incoming data, converting types when appropriate, and providing clear error messages when validation fails.

<aside>
💡

Python’s dynamic typing gives programmers fast development and freedom, but this comes at the cost of **introducing type validation issues** that often surface in production.

</aside>

# **What Is Pydantic?**

when you build applications that interact with the outside world. Users might *submit a phone number as a string instead of an integer*, an *API might return null values where you expect numbers*, or a *configuration file might contain typos* that break your application at 3 AM.

=> Pydantic solves this by combining three powerful concepts: type hints, runtime validation, and automatic serialization

you define your data structure once using Python’s type annotation syntax, and Pydantic handles all the validation **automatically**:

```python
from pydantic import BaseModel, EmailStr
from typing import Optional

class User(BaseModel):
   age: int
   email: EmailStr
   is_active: bool = True
   nickname: Optional[str] = None

# Pydantic automatically validates and converts data
user_data = {
   "age": "25",  # String gets converted to int
   "email": "john@example.com",
   "is_active": "true"  # String gets converted to bool
}

user = User(**user_data)
print(user.age)  # 25 (as integer)
print(user.model_dump())  # Clean dictionary output
```

# **Pydantic’s features**

Pydantic also allows integration with modern frameworks, **FastAPI.**  When you define a Pydantic model, you get **OpenAPI** schema generation for free:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class UserCreate(BaseModel):
   name: str
   email: EmailStr
   age: int

@app.post("/users/")
async def create_user(user: UserCreate):
   # FastAPI automatically validates the request body
   # and generates API docs from your Pydantic model
   return {"message": f"Created user {user.name}"}
```

⇒ JSON schema generation happens automatically with every Pydantic model. This means your data structures become self-documenting, and you can generate client libraries, validation rules for frontend applications, or database schemas from the same source of truth.

## **Your first Pydantic model**

```python
from pydantic import BaseModel, EmailStr
from typing import Optional
from datetime import datetime

class User(BaseModel):
   name: str
   email: EmailStr
   age: int
   is_active: bool = True
   created_at: datetime = None

# Test with clean data
clean_data = {
   "name": "Alice Johnson",
   "email": "alice@example.com",
   "age": 28
}

user = User(**clean_data)
print(f"User created: {user.name}, Age: {user.age}")
print(f"Model output: {user.model_dump()}")
```

When validation fails, Pydantic provides clear error messages:

```python
from pydantic import ValidationError

try:
   invalid_user = User(
       name="",  # Empty string
       email="not-an-email",  # Invalid email
       age=-5  # Negative age
   )
except ValidationError as e:
   print(e)
```

# **Building Data Models With Pydantic**

## **Field validation and constraints**

Consider a product catalog API where price data comes from multiple vendors with *different formatting standards*. Some send prices as strings, others as floats, and occasionally, someone sends a negative price that crashes your billing system.

Pydantic’s `Field()` function transforms basic type hints into sophisticated validation rules that protect your application:

```python
from pydantic import BaseModel, Field
from decimal import Decimal
from typing import Optional

class Product(BaseModel):
   name: str = Field(min_length=1, max_length=100)
   price: Decimal = Field(gt=0, le=10000)  # Greater than 0, less than or equal to 10,000
   description: Optional[str] = Field(None, max_length=500)
   category: str = Field(..., pattern=r'^[A-Za-z\s]+$')  # Only letters and spaces
   stock_quantity: int = Field(ge=0)  # Greater than or equal to 0
   is_available: bool = True

# This works - all constraints satisfied
valid_product = Product(
   name="Wireless Headphones",
   price="199.99",  # String converted to Decimal
   description="High-quality wireless headphones",
   category="Electronics",
   stock_quantity=50
)

# This fails with clear error messages
try:
   invalid_product = Product(
       name="",  # Too short
       price=-50,  # Negative price
       category="Electronics123",  # Contains numbers
       stock_quantity=-5  # Negative stock
   )
except ValidationError as e:
   print(f"Validation errors: {len(e.errors())} issues found")
```

 `min_length` and `max_length` prevent database schema violations

 `gt` and `le` create business logic boundaries

`pattern` validates formatted data using regular expressions. 

The `Field(...)` syntax with ellipsis marks the required fields, while `Field(None, ...)` creates optional fields with validation rules.

<aside>
💡

`Field(...)` = **REQUIRED** field, you must provide a value
ex: User(): fail / str = Field(…, pattern = r’smth smth’): work

 `Field(None, ...)` = **OPTIONAL** field

ex: User(): work/ Optional[str] = Field(None,…): work

</aside>

## **Type coercion vs strict validation**

By default, Pydantic converts compatible types rather than rejecting them outright. This flexibility works well for user input, but some scenarios demand **exact type matching**

```python
from pydantic import BaseModel, Field, ValidationError

# Default: lenient type coercion
class FlexibleOrder(BaseModel):
   order_id: int
   total_amount: float
   is_paid: bool

# These all work due to automatic conversion
flexible_order = FlexibleOrder(
   order_id="12345",  # String to int
   total_amount="99.99",  # String to float
   is_paid="true"  # String to bool
)

# Strict validation when precision matters
class StrictOrder(BaseModel):
   model_config = {"str_strip_whitespace": True, "validate_assignment": True}
  
   order_id: int = Field(strict=True)
   total_amount: float = Field(strict=True)
   is_paid: bool = Field(strict=True)
```

The `model_config` dictionary controls validation behavior across your entire model. 

The `str_strip_whitespace` option cleans string input automatically

`validate_assignment` ensures field changes after model creation still trigger validation

Individual fields can override these settings with `Field(strict=True)` for situations requiring exact type matching, like financial calculations or scientific data.

# **Nested models and complex data**

```python
from typing import List
from datetime import datetime

class Address(BaseModel):
   street: str = Field(min_length=5)
   city: str = Field(min_length=2)
   postal_code: str = Field(pattern=r'^\d{5}(-\d{4})?$')
   country: str = "USA"

class Customer(BaseModel):
   name: str = Field(min_length=1)
   email: EmailStr
   shipping_address: Address
   billing_address: Optional[Address] = None

class OrderItem(BaseModel):
   product_id: int = Field(gt=0)
   quantity: int = Field(gt=0, le=100)
   unit_price: Decimal = Field(gt=0)

class Order(BaseModel):
   order_id: str = Field(pattern=r'^ORD-\d{6}$')
   customer: Customer
   items: List[OrderItem] = Field(min_items=1)
   order_date: datetime = Field(default_factory=datetime.now)

# Complex nested data validation
order_data = {
   "order_id": "ORD-123456",
   "customer": {
       "name": "John Doe",
       "email": "john@example.com",
       "shipping_address": {
           "street": "123 Main Street",
           "city": "Anytown",
           "postal_code": "12345"
       }
   },
   "items": [
       {"product_id": 1, "quantity": 2, "unit_price": "29.99"},
       {"product_id": 2, "quantity": 1, "unit_price": "149.99"}
   ]
}

order = Order(**order_data)
print(f"Order validated with {len(order.items)} items")
```

# **Optional fields and None handling**

```python
from typing import Optional

class UserCreate(BaseModel):
   name: str = Field(min_length=1)
   email: EmailStr
   age: int = Field(ge=13, le=120)
   phone: Optional[str] = Field(None, pattern=r'^\+?1?\d{9,15}$')

class UserUpdate(BaseModel):
   name: Optional[str] = Field(None, min_length=1)
   email: Optional[EmailStr] = None
   age: Optional[int] = Field(None, ge=13, le=120)
   phone: Optional[str] = Field(None, pattern=r'^\+?1?\d{9,15}$')

# PATCH request with partial data
update_data = {"name": "Jane Smith", "age": 30}
user_update = UserUpdate(**update_data)

# Serialize only provided fields
patch_data = user_update.model_dump(exclude_none=True)
print(f"Fields to update: {list(patch_data.keys())}")
```

*Serialization* converts Pydantic objects back into dictionaries or JSON strings for storage or transmission ⇒ model_dump() ⇒ works perfectly for PATCH requests where clients send only the fields they want to change

> 👉 Serialization = **turning a Python object into a format that can be stored or sent**
UserUpdate = Python object, `model_dump()` transform it to JSON / dict
> 

# **Custom Validation and Real-World Integration**

## **Field validators and model validation**

When business logic determines data validity, Pydantic’s `@field_validator` decorator transforms your validation functions into part of the model itself

```python
from pydantic import BaseModel, field_validator, Field
import re

class UserRegistration(BaseModel):
   username: str = Field(min_length=3)
   email: EmailStr
   password: str
   subscription_tier: str = Field(pattern=r'^(free|pro|enterprise)$')
  
   @field_validator('password') #Run this function when validating the password field
   @classmethod
   def validate_password_complexity(cls, password, info):
       tier = info.data.get('subscription_tier', 'free')
 #Get the value of subscription_tier that was already validated, or use 'free' if it doesn't exist
      
       if len(password) < 8:
           raise ValueError('Password must be at least 8 characters')
          
       if tier == 'enterprise' and not re.search(r'[A-Z]', password):
           raise ValueError('Enterprise accounts require uppercase letters')
          
       return password
```

The `@field_validator` decorator gives you access to other field values through the `info.data` parameter, allowing validation rules that depend on multiple fields. The validator runs after basic type checking passes, so you can safely assume the `subscription_tier` is one of the allowed values.

> 👉 `info` = **validation context object** (field values, config, field metadata)

👉 `info.data` = **dictionary of already parsed/validated fields**

👉 `cls` refers to Class (User)
> 

For validation that spans multiple fields, the `@model_validator` decorator runs after all individual fields are validated:

```python
from datetime import datetime
from pydantic import model_validator

class EventRegistration(BaseModel):
   start_date: datetime
   end_date: datetime
   max_attendees: int = Field(gt=0)
   current_attendees: int = Field(ge=0)
  
   @model_validator(mode='after')
   def validate_event_constraints(self):
       if self.end_date <= self.start_date:
           raise ValueError('Event end date must be after start date')
          
       if self.current_attendees > self.max_attendees:
           raise ValueError('Current attendees cannot exceed maximum')
          
       return self
```

The `mode='after'` : Run **AFTER** all fields are validated and the object is created

> The validator **MUST** return `self` to indicate successful validation.
👉 `self` = **the fully created model instance**
> 

## **FastAPI integration**

The key pattern involves creating separate models for different operations, giving you control over what data flows in each direction:

```python
from fastapi import FastAPI
from typing import Optional
from datetime import datetime

app = FastAPI()

class UserCreate(BaseModel):
   username: str = Field(min_length=3)
   email: EmailStr
   password: str = Field(min_length=8)

class UserResponse(BaseModel):
   id: int
   username: str
   email: EmailStr
   created_at: datetime
  
@app.post("/users/", response_model=UserResponse)
async def create_user(user: UserCreate):
   # FastAPI automatically validates the request body
   new_user = {
       "id": 1,
       "username": user.username,
       "email": user.email,
       "created_at": datetime.now()
   }
   return UserResponse(**new_user) #strong typing, fastAPI gon transform to JSON regardless
#response_model ensures the outcome must match the schema (type validaion, etc)
```

For update operations, you can create models where all fields are optional:

```python
class UserUpdate(BaseModel):
   username: Optional[str] = Field(None, min_length=3)
   email: Optional[EmailStr] = None

fake_db = {
    1: {"username": "old", "email": "old@email.com"}
}

@app.patch("/users/{user_id}")
async def update_user(user_id: int, user_update: UserUpdate #fastAPI parses the input client sent):
   # Only update provided fields
   update_data = user_update.model_dump(exclude_unset=True)
   
	# get existing user
   user = fake_db.get(user_id)

    # update only provided fields
   user.update(update_data)
    
   return {"message": f"Updated user {user_id}"}
```

The `exclude_unset=True` parameter in PATCH operations ensures you only update fields that were explicitly provided, preventing accidental overwrites

## **Configuration management with environment variables**

Production applications need secure, deployment-friendly configuration management. Pydantic’s `BaseSettings` combined with `.env` files provides type-safe configuration that works across development, staging, and production environments.

First, create a `.env` file in your project root:

```python
# .env file
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
SECRET_KEY=your-secret-key-here
DEBUG=false
ALLOWED_HOSTS=localhost,127.0.0.1,yourdomain.com
```

Then define your settings model:

```python
from pydantic import BaseSettings, Field
from typing import List

class AppSettings(BaseSettings):
   database_url: str = Field(description="Database connection URL")
   secret_key: str = Field(description="Secret key for JWT tokens")
   debug: bool = Field(default=False)
   allowed_hosts: List[str] = Field(default=["localhost"])
  
   class Config: #configuration blueprint that Pydantic reads auto, no need to instan
       env_file = ".env"
       case_sensitive = False

# Load settings automatically from environment and .env file
settings = AppSettings()
```

The `BaseSettings` class automatically reads from environment variables, `.env` files, and command-line arguments. 

Environment variables take precedence over `.env` file values, making it easy to override settings in different deployment environments. 

The `case_sensitive = False` setting allows flexible environment variable naming.

For complex applications, you can organize settings into logical groups:

```python
class DatabaseSettings(BaseSettings):
   url: str = Field(env="DATABASE_URL")
   max_connections: int = Field(default=5, env="DB_MAX_CONNECTIONS")

class AppSettings(BaseSettings):
   secret_key: str
   debug: bool = False
   database: DatabaseSettings = DatabaseSettings()
  
   class Config:
       env_file = ".env"

settings = AppSettings()
# Access nested configuration
db_url = settings.database.url
```