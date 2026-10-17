Section 1: Architecture Summary

The Smart Clinic Management System is engineered using a robust three-tier architecture powered by Spring Boot, combining server-side rendering with Thymeleaf and decoupled RESTful APIs. The presentation layer splits responsibilities between traditional MVC controllers rendering dynamic HTML dashboards and REST controllers serving JSON payloads for modern clients. The business logic resides within dedicated service components that manage transaction boundaries and coordinate operations across a polyglot persistence (dual database) strategy. Relational data requiring strict ACID compliance and structured relationships—such as patient demographics, staff management, and billing—is handled by MySQL via Spring Data JPA. Simultaneously, semi-structured or high-volume clinical data—such as detailed medical history logs, doctor consultation notes, and lab reports—is stored in MongoDB via Spring Data MongoDB. This decoupled, layered design ensures high scalability, maintainability, and clean separation of concerns across the application lifecycle.

Section 2: Numbered Flow

The request and response cycle flows differently depending on whether the interaction originates from the internal web dashboard (Thymeleaf) or an external client consuming the REST API.

Flow A: Web Dashboard Request (Thymeleaf MVC)

1.User Interaction: The user interacts with the browser UI (e.g., clicking a link to view patient appointments on the dashboard), triggering an HTTP request.

2.DispatcherServlet Reception: The HTTP request reaches the Spring Boot application and is intercepted by the front controller, DispatcherServlet.

3.Controller Routing: DispatcherServlet inspects the URL mapping and routes the request to the appropriate Spring MVC @Controller.

4.Service Invocation: The MVC controller invokes the corresponding business method within the @Service layer.

5.Data Access (Dual Database Coordination):

    
  The service calls Spring Data JPA repositories to query relational data from MySQL (e.g., fetching patient and doctor records).

  If applicable, the service simultaneously calls Spring Data MongoDB repositories to fetch document-based clinical logs or notes from MongoDB.

6.Data Processing: The repositories execute queries against their respective databases, returning entity objects and documents back to the service layer.

7.Model Population: The service layer processes the data and returns domain models to the MVC controller. The controller populates a Spring Model object.

8.View Rendering: The controller returns a view name (e.g., appointments-dashboard). The Thymeleaf engine parses the template, injecting the model data into the HTML structure.

9.HTTP Response: The rendered HTML page is returned through the DispatcherServlet as an HTTP response to the user's browser.

Flow B: RESTful API Request (JSON)

  1.Client Request: An external client or frontend application sends an HTTP request (e.g., GET /api/v1/patients) with JSON headers to the server.

  2.DispatcherServlet & REST Controller: The request hits the DispatcherServlet, which routes it to the matching @RestController.

  3. Service Layer Execution: The @RestController delegates business logic execution to the appropriate @Service component.

  4. Repository Interaction: The service layer queries MySQL via Spring Data JPA or MongoDB via Spring Data MongoDB to fetch or persist data.

  5.Data Serialization: The database returns data objects/documents to the service, which passes them back to the REST controller.

  5.JSON Response: Spring Boot's default message converter (Jackson) automatically serializes the Java objects into a JSON payload and returns an HTTP response with status 200 OK (or appropriate status codes) to the client.
