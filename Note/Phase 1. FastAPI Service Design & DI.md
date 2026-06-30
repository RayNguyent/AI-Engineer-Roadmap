*Dependency Injection*

**Dependency Injection (DI)** is a design pattern where an object receives its dependencies from an external source instead of creating them itself. It separates dependency creation from usage, improving flexibility, testability, and maintainability.

- Reduces tight coupling between classes while improving code reusability and flexibility.
- Makes unit testing easier using mock dependencies and enhances system maintainability and scalability.

<aside>
💡

**Example:** `Car` class might depend on a `Engine` class to run. Without DI, the `Car` class would directly create or manage the `Engine` instance within its code, which makes the two classes tightly coupled. This approach can create problems, particularly when you need to *test*, *extend*, or *modify* the classes in the future.

</aside>

- Dependency Injection solves this problem by **injecting** the dependencies (like the `Engine` ones in the `Car` example) into the class **from an external source**, rather than having the class create them.

⇒ DI allows you to "inject" the things a class needs (its dependencies) from the outside, instead of letting the class create or manage them itself.

## **Four Roles of Dependency Injection**

!image.png

### **1. Client**

The client is the class or component that depends on a service to perform its operations.

- It does **NOT** create or manage its dependencies
- It **receives required services** from the injector

Example:
A FastAPI route handler that receives a `ChatService` instance.

### **2. Service**

The service is the class or component that provides specific functionality needed by the client.

- It contains the **actual business logic**
- It is designed to be **independent of the client**

⇒  The service is something the client **needs**, but does not create.

### **3. Injector**

The injector is responsible for creating service instances and supplying them to the client.

- It manages **dependency creation and lifecycle**
- It injects required services at runtime
- The injector is usually the DI system (`Depends`) or your own DI container.
- FastAPI itself acts as the injector when resolving dependencies.

### **4. Interface**

The interface defines a contract that specifies what methods a service must implement.

- **Clients depend on interfaces**, not concrete classes
- It allows easy replacement of service implementations

⇒ defines *what the service must be able to do*.

## 🏎️ Analogy: A Professional Racing Team

Think of your DI workflow as a **Formula 1 racing team** preparing a car for a race.

### 1. The Injector → The Pit Crew Chief

The **pit crew chief** is responsible for:

- Selecting the right engine
- Installing it into the car
- Ensuring the car receives the correct components

The chief **does NOT drive the car**. They only **prepare** it.

### **2. The Service → The Engine**

The **engine** is the component that actually performs the work:

- Generates power
- respond to the driver’s commands
- replaceable

The engine doesn’t know:

- Who the driver is
- Who installed it
- What race it will run

It just does its job.

⇒ ex: db gateway, LLM client, Business logic service

=⇒ Pit Crew Chief select engine → PCC installs the engine into the car → Driver gets in the car (client receives the dependency) → Drivers races (uses the service) → Engine performs the work (aka service executes logic)

### 3. The Client → The Driver

The **driver** is the one who:

- Operates the car
- Uses the engine
- Executes the race strategy

But the driver **does NOT install the engine**. They simply **receive** the car already prepared.

# **Uses**

- **Loose Coupling & Reusability:** Objects don’t create their own dependencies, making them independent and reusable.
- **Improved Testability:** Easily inject mocks or test doubles to test components in isolation.
- **Maintainability & Flexibility:** DI frameworks simplify managing and configuring dependencies.
- **Scalability & Extensibility:** Helps handle complex dependency graphs in large applications.
- **Cross-Cutting Concerns:** Conveniently inject services like logging, security, or caching across multiple components.

```python
from EmailProvider import EmailProvider

class NotificationService:
    def __init__(self):
        self.email_provider = EmailProvider()

    def send_notification(self, message, recipient):
        self.email_provider.send_email(message, recipient)
        
#Tight Coupling: The *NotificationService* is tightly coupled to the EmailProvider, making it difficult to switch to a different provider without code changes.
#Testability: Testing NotificationService in isolation is challenging as it directly uses EmailProvider.
```

```python
# interfaces.py
from abc import ABC, abstractmethod

class NotificationProvider(ABC):
    @abstractmethod
    def sendNotification(self, message: str, recipient: str):
        pass

class EmailProvider(NotificationProvider):
    def sendNotification(self, message: str, recipient: str):
        print(f"Sending Email to {recipient}: {message}")

class SMSProvider(NotificationProvider):
    def sendNotification(self, message: str, recipient: str):
        print(f"Sending SMS to {recipient}: {message}")
       
# services.py
from interfaces import NotificationProvider

class NotificationService:
    def __init__(self, provider: NotificationProvider):
        self.provider = provider

    def send(self, message: str, recipient: str):
        self.provider.sendNotification(message, recipient)

# di.py
from fastapi import Depends
from interfaces import EmailProvider, SMSProvider, NotificationProvider
from services import NotificationService

# Choose which provider to inject
def get_provider() -> NotificationProvider:
    # Swap EmailProvider() → SMSProvider() to change behavior
    return EmailProvider()

def get_notification_service(
    provider: NotificationProvider = Depends(get_provider)
) -> NotificationService:
    return NotificationService(provider)
    
    
# schemas.py
from pydantic import BaseModel

class NotificationRequest(BaseModel):
    message: str
    recipient: str

class NotificationResponse(BaseModel):
    status: str

# main.py
from fastapi import FastAPI, Depends
from schemas import NotificationRequest, NotificationResponse
from di import get_notification_service
from services import NotificationService

app = FastAPI()

@app.post("/notify", response_model=NotificationResponse) #Whenever a POST request hits 
																						    #/notify, call the notify() function
def notify(
    req: NotificationRequest,
    service: NotificationService = Depends(get_notification_service)
):
    service.send(req.message, req.recipient)
    return NotificationResponse(status="Notification sent")
#endpoint: func, defines what the API accepts and returns

```

### **Injector → Client**

FastAPI injects:

- `provider = EmailProvider()`
- `service = NotificationService(provider)`

### **Client → Service**

The endpoint receives `service` via `Depends`.

### **Service → Provider**

`NotificationService` calls the provider’s implementation.

### **Provider → Work**

Email/SMS provider prints the message.

<aside>
💡

An endpoint is like a **front desk** at a company:

- Clients walk in through a specific door (`/notify`)
- They hand you a form (`NotificationRequest`)
- The front desk calls the right internal department (`NotificationService`)
- The department does the work
- The front desk gives the client a structured response (`NotificationResponse`)
</aside>