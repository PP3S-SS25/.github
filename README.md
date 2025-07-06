![logo-quer](https://github.com/user-attachments/assets/eaf55c84-056c-4ac6-8575-f018d9392f28)

## 1 Task

Our task is to create a campus path-finding application that computes routes under 30 minutes, including intermediate stops like cafés, restaurants, or toilets. It will integrate TU Berlin location maps, personal insider knowledge (special paths), and external APIs (e.g., BVG) to provide accurate and user-relevant routing information.

## 2 Approach

Our approach focuses on applying industry-relevant tools and standards to closely mirror real-world workflows and best practices (e.g. secure storage of passwords). The team was divided into smaller subteams of two, each taking ownership of a specific component and communicating strictly via well-defined APIs to enforce _Separation of Concerns_. This structure encourages accountability and ensures that each team is fully responsible for the functionality and quality of their own component.
By working with unfamiliar technologies — such as new programming languages, specialized databases, and modern frameworks — team members were encouraged to step outside their comfort zones and expand both their technical and collaborative skills.
Furthermore, we focused on designing all system components to be as loosely coupled as possible. This architectural choice supports long-term maintainability, allowing the system to evolve with minimal refactoring while enabling straightforward integration of new features.

## 3 Architecture

<img width="1004" alt="image" src="https://github.com/user-attachments/assets/7ca10621-566d-4c22-bb79-ed588279d31f" />

The architecture of our system is composed of three main components: a **React-based frontend** for user interaction, a **Go-based API Gateway** that manages request translation and external API integration, and a **Rust-based backend** that performs routing using geospatial queries on a PostGIS-enhanced PostgreSQL database. Each layer is designed for clear separation of concerns, enabling modular development, efficient communication, and low-latency route computation.
### 3.1: Frontend
The user interface is built with the widely adopted JavaScript framework React, extended with TypeScript for type safety and better maintainability. At its core is a Leaflet-based map, displaying the TU Berlin campus and offering a search bar to enter the desired destination. Destinations can be TU buildings, cafés, or restaurants. The starting point can either be one of these or be determined via the browser's Geolocation API.
##### 3.1.1 Tech Stack
React was chosen for its popularity, strong ecosystem, and relevance in modern web development. It simplifies UI creation through JSX, allowing HTML-like code within JavaScript, and encourages reusable, modular code via components.

For map rendering, we use Leaflet, a lightweight and flexible open-source library that integrates smoothly and leverages OpenStreetMap data.

##### 3.1.2 Backend Integration
On page load, all available buildings (TU buildings, cafés, and restaurants) are fetched from the backend and visualized as markers on the map. Users can click on any of these locations to select them as a starting point or destination.
##### 3.1.3 User Journey
Upon visiting the website, the user's current location is requested (if permission is granted) and marked on the map with a red marker. The user can then select a destination from the listed locations provided by the backend.

Once a destination is chosen, the routing menu expands. The user selects a starting point - either their current location or another building.

Optionally, intermediate stops can be added. After clicking “Go”, the routing request is sent to the backend, and the calculated route is displayed on the map using highlighted paths.
##### 3.1.4 User Accounts
Users can create an account or log in by expanding the right-hand sidebar and being redirected to the login page. Once logged in, users can access their favorite destinations and directly start routing to them.

### 3.2: API Gateway
The component serves as a middleware layer that decouples the frontend and backend server, offering two key advantages:  
First, it enforces loose coupling between components. Both the frontend and backend maintain their own internal data models and requirements, remaining independent of each other's implementation details. The API Gateway bridges the gap by converting and relaying data in formats suited to each side - JSON for the frontend and Protocol Buffers for the backend.

Second, in alignment with our **Separation of Concerns** approach, we avoid overloading the backend server with tasks beyond its core functionality. Managing external API calls and client communication would have blended responsibilities and added unnecessary complexity. Introducing the API Gateway allows us to keep responsibilities and data formats clearly separated within our architecture.

##### 3.2.1 Language Choice

We selected **Go (Golang)** as the implementation language for the API Gateway due to its strong support for web-related development and seamless integration of HTTP and gRPC functionalities. Although both team members were initially unfamiliar with Go, its concise syntax and rich standard library enabled a fast learning curve and productive development. Moreover, Go's support for both raw JSON handling and gRPC communication made it an ideal fit for our component's bridging role.

##### 3.2.2 External API Integration: BVG and Weather

To meet our routing goals across the TU Berlin campus - which spans a considerably large area - and to determine the truly fastest route, we integrated two external data sources: BVG and weather data.

Switching lecture halls across distant campus buildings within a 30-minute break can be quite challenging. Therefore, BVG's real-time public transport data was used to include the connections on lines _U2, 245_ and _M45_ which serve major campus streets such as _Marchstraße_ and _Hardenbergstraße_. Unfortunately, no line connects _Straße des 17. Juni_ which had significantly improved fast transit across the campus.

Additionally, given that Berlin can be very cold and rainy during fall and winter, we aimed to minimize exposure to outdoor conditions. The API gateway fetches live weather data and calculates a weather index, which influences the routing logic in the backend (see section 4).

##### 3.2.3 Inter-Component Communication

Each core functionality exposed to the frontend corresponds to a dedicated HTTP endpoint in the API Gateway. The communication flow is as follows:

1. The frontend submits a JSON-formatted request to the API Gateway.
2. The data is converted and, along with additional API information, forwarded to the backend using gRPC and Protocol Buffers to ensure type safety and efficient serialization.
3. After the backend responds, the result is converted back to JSON and returned to the frontend.

This structure allows us to maintain a well-defined interface between frontend and backend while optimizing performance and flexibility in data handling.
### 3.3: Ressource Management Layer

The Resource Management Layer consists of a gRPC server component written in Rust that handles incoming requests. It accesses the database, which stores OpenStreetMap (OSM) data and performs routing internally. To ensure low latency, a caching layer has been implemented in front of the database.
##### Section 3.3.1: Language Choice
Rust was chosen due to its blazingly fast performance once compiled, enabling us to push towards very low latencies. Additionally, Rust ensures memory safety without relying on a garbage collector. It also aligns with our educational goal of working in an unfamiliar environment, thanks to Rust’s unique and expressive problem-solving style. 
##### Section 3.3.2: Database and Geospatial Support
All data is stored in a PostgreSQL database extended with PostGIS and several other extension like pgRouting and hstore, which are specifically tailored for geospatial data processing.
PostGIS is one of the most relevant tools in Geospatial Software with corporate sponsors like (Palantir Technologies, Regione Toscana and the U.S. Department of State and many more)
This set of tools enables storage, indexing and querying of geospatial data directly in the database. And allows for complex spatial operations like distance calculations, area measurements and spatial joins. 
##### Section 3.3.3: Routing within the Database
As mentioned in Section 3.2 we also extended the PostGIS database with a specific framework: *pgRouting* which adds network and routing functionalities directly to the database. This framework provides algorithms like *Dijkstra, A\** for shortest path, driving distance and cost-based routing. We will cover Dijkstra and cost-based routing later in Section 4.
This approach allows routing directly on the dataset without moving data out of the database.

##### Section 3.3.4: Caching Layer
To even futher minimize the latency of a call, we introduced a Valkey (Redis fork) instance which is used to cache requested routes for an hour. In addition, precomputed building-to-building routes are persisted in a dedicated table further improve lookup speed.

## 4 Routing 

Our routing system is based on OpenStreetMap data, using a subset of the data of Berlin which covers the Technical University of Berlin. We use osm2pgrouting for preproccesses while loading the data into the database to enable routing within the database.

### 4.1 Routing Features
Our system supports a variety of routing types:
**Predefined Building-to-Building Routing**: Users can select buildings from a predefined set. If a route between them has already been calculated, it is retrieved from a cache or a dedicated building-to-building routes table. Otherwise, the system computes the route on demand, stores it for future use, and caches it, ensuring fast startup times and avoiding unnecessary precomputation of unused routes.

**Intermediate Waypoint**: Users can add multiple intermediate stops, including buildings or amenities. Requests are split at the API Gateway and sent in parallel to the Resource Management Layer Overlord for processing.

**Public Transport Integration**: Bus routes are integrated using OSM-derived data, with bus stops as transfer nodes. We do not recalculate the bus path itself but filter the existing route to include only the relevant segments between the chosen stops.
### 4.2 Routing Algorithm and Weather-Dependent Weighting
We use **Dijkstra's algorithm** via **pgRouting** (`pgr_dijkstra()`). Outdoor path weights are dynamically adjusted with a weather penalty (factor 1 for good weather, 2 for bad weather), making them less attractive when conditions worsen. Real-time weather data is provided by the middleware and passed to the Database Overlord to update routing weights on demand.


## 5 Challenges and Lessons Learned
For most of us, was this project the first project within a larger group. So we had several challenges we had to come across, both technical and team wise.

### 5.1 Technical Challenges
The team encountered a broad range of technical hurdles due to the project’s complexity and diverse tech stack. Many members worked with technologies they had little or no prior experience with, such as Go for the API Gateway, Rust for the backend, and advanced frontend tools like React, TypeScript, and CSS. This unfamiliarity initially slowed productivity and led to trial-and-error-based development.  
Integrating various layers of the application—from frontend to API Gateway to backend—was especially challenging, as it required consistent data handling across JSON, gRPC, and PostGIS types. Handling complex routing logic added another layer of difficulty, such as determining nearest bus stops connected by the same line or interpreting non-deterministic routing outputs from the BVG API. Furthermore, geospatial data manipulation using PostGIS and coordinate handling (latitude/longitude inconsistencies) presented persistent friction. Merge conflicts and the overwhelming number of frontend libraries/tools added to the challenge.

### 5.2 Teamwork Challenges
Beyond technical aspects, the team struggled with communication and coordination. With multiple subteams working on different components, maintaining a shared understanding of the product vision proved difficult. Breaking down tasks clearly and avoiding duplicate work was a recurring issue. Some team members also expressed initial insecurity about their coding abilities and lack of web development experience, which impacted early progress.  
Leadership and group dynamics emerged as a hidden challenge: keeping track of progress across all teams, aligning development timelines, and ensuring everyone moved toward the same goal required more active management than anticipated. Despite these challenges, we noticed growth in technical confidence and collaborative skills over time.


## 6 On-going work

As a next step, we plan to improve even more the request latency by adding insider knowledge to the OSM data, such as indoor pathways. These enhancements will also be contributed back to the open-source OSM project, benefiting future applications built on top of it.
