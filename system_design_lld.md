# Low-Level Design (LLD) Guide: Object-Oriented Design & Design Patterns

## Table of Contents

1. [Introduction to Low-Level Design](#introduction-to-low-level-design)
2. [SOLID Principles](#solid-principles)
3. [Design Patterns](#design-patterns)
4. [Real-World LLD Examples](#real-world-lld-examples)
5. [Best Practices](#best-practices)

---

## Introduction to Low-Level Design

Low-Level Design (LLD) focuses on the detailed design of individual components, including:
- Class diagrams and relationships
- Object-oriented design principles
- Design patterns
- Code organization
- Module interfaces

### Key Objectives
- **Maintainability**: Easy to modify and extend
- **Reusability**: Components can be reused
- **Testability**: Easy to unit test
- **Flexibility**: Adapt to changing requirements
- **Readability**: Clear and understandable code

---

## SOLID Principles

### 1. Single Responsibility Principle (SRP)

**"A class should have only one reason to change"**

❌ **Bad Example:**
```java
public class User {
    private String name;
    private String email;
    
    // User management
    public void save() {
        // Database logic
        Connection conn = DriverManager.getConnection("...");
        // SQL queries...
    }
    
    // Email sending
    public void sendEmail(String message) {
        // Email sending logic
        SMTPConnection smtp = new SMTPConnection();
        // Send email...
    }
    
    // Validation
    public boolean validateEmail() {
        // Validation logic
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
}
```

✅ **Good Example:**
```java
// Single responsibility: User data
public class User {
    private String name;
    private String email;
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // Getters and setters only
    public String getName() { return name; }
    public String getEmail() { return email; }
}

// Single responsibility: User persistence
public class UserRepository {
    public void save(User user) {
        Connection conn = DriverManager.getConnection("...");
        // SQL queries...
    }
    
    public User findById(int id) {
        // Fetch from database
    }
}

// Single responsibility: Email operations
public class EmailService {
    public void sendEmail(String to, String message) {
        SMTPConnection smtp = new SMTPConnection();
        // Send email...
    }
}

// Single responsibility: Validation
public class UserValidator {
    public boolean validateEmail(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
}
```

### 2. Open/Closed Principle (OCP)

**"Open for extension, closed for modification"**

❌ **Bad Example:**
```java
public class DiscountCalculator {
    public double calculateDiscount(String customerType, double amount) {
        if (customerType.equals("Regular")) {
            return amount * 0.05;
        } else if (customerType.equals("Premium")) {
            return amount * 0.10;
        } else if (customerType.equals("VIP")) {
            return amount * 0.20;
        }
        return 0;
    }
}
// Adding new customer type requires modifying this class
```

✅ **Good Example:**
```java
// Abstract discount strategy
public interface DiscountStrategy {
    double calculateDiscount(double amount);
}

public class RegularCustomerDiscount implements DiscountStrategy {
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.05;
    }
}

public class PremiumCustomerDiscount implements DiscountStrategy {
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.10;
    }
}

public class VIPCustomerDiscount implements DiscountStrategy {
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.20;
    }
}

public class DiscountCalculator {
    private DiscountStrategy strategy;
    
    public DiscountCalculator(DiscountStrategy strategy) {
        this.strategy = strategy;
    }
    
    public double calculateDiscount(double amount) {
        return strategy.calculateDiscount(amount);
    }
}

// Usage - can add new discount types without modifying existing code
DiscountCalculator calculator = new DiscountCalculator(new VIPCustomerDiscount());
double discount = calculator.calculateDiscount(1000);
```

### 3. Liskov Substitution Principle (LSP)

**"Derived classes must be substitutable for their base classes"**

❌ **Bad Example:**
```java
public class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getArea() {
        return width * height;
    }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width; // Violates LSP
    }
    
    @Override
    public void setHeight(int height) {
        this.width = height;  // Violates LSP
        this.height = height;
    }
}

// Problem:
Rectangle rect = new Square();
rect.setWidth(5);
rect.setHeight(10);
// Expected: 50, Actual: 100 (Square behavior breaks Rectangle contract)
```

✅ **Good Example:**
```java
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private int width;
    private int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public int getArea() {
        return width * height;
    }
}

public class Square implements Shape {
    private int side;
    
    public Square(int side) {
        this.side = side;
    }
    
    @Override
    public int getArea() {
        return side * side;
    }
}
```

### 4. Interface Segregation Principle (ISP)

**"Clients should not be forced to depend on interfaces they don't use"**

❌ **Bad Example:**
```java
public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class HumanWorker implements Worker {
    @Override
    public void work() {
        System.out.println("Human working");
    }
    
    @Override
    public void eat() {
        System.out.println("Human eating");
    }
    
    @Override
    public void sleep() {
        System.out.println("Human sleeping");
    }
}

public class RobotWorker implements Worker {
    @Override
    public void work() {
        System.out.println("Robot working");
    }
    
    @Override
    public void eat() {
        // Robots don't eat - forced to implement
        throw new UnsupportedOperationException();
    }
    
    @Override
    public void sleep() {
        // Robots don't sleep - forced to implement
        throw new UnsupportedOperationException();
    }
}
```

✅ **Good Example:**
```java
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class HumanWorker implements Workable, Eatable, Sleepable {
    @Override
    public void work() {
        System.out.println("Human working");
    }
    
    @Override
    public void eat() {
        System.out.println("Human eating");
    }
    
    @Override
    public void sleep() {
        System.out.println("Human sleeping");
    }
}

public class RobotWorker implements Workable {
    @Override
    public void work() {
        System.out.println("Robot working");
    }
    // No need to implement eat() or sleep()
}
```

### 5. Dependency Inversion Principle (DIP)

**"Depend on abstractions, not concretions"**

❌ **Bad Example:**
```java
public class MySQLDatabase {
    public void connect() {
        System.out.println("Connected to MySQL");
    }
    
    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}

public class UserService {
    private MySQLDatabase database;
    
    public UserService() {
        this.database = new MySQLDatabase(); // Tight coupling
    }
    
    public void getUser(int id) {
        database.connect();
        database.query("SELECT * FROM users WHERE id = " + id);
    }
}
// Switching to PostgreSQL requires modifying UserService
```

✅ **Good Example:**
```java
// Abstraction
public interface Database {
    void connect();
    void query(String sql);
}

// Concrete implementations
public class MySQLDatabase implements Database {
    @Override
    public void connect() {
        System.out.println("Connected to MySQL");
    }
    
    @Override
    public void query(String sql) {
        System.out.println("MySQL executing: " + sql);
    }
}

public class PostgreSQLDatabase implements Database {
    @Override
    public void connect() {
        System.out.println("Connected to PostgreSQL");
    }
    
    @Override
    public void query(String sql) {
        System.out.println("PostgreSQL executing: " + sql);
    }
}

// Depend on abstraction
public class UserService {
    private Database database;
    
    public UserService(Database database) {
        this.database = database; // Dependency injection
    }
    
    public void getUser(int id) {
        database.connect();
        database.query("SELECT * FROM users WHERE id = " + id);
    }
}

// Usage
Database mysql = new MySQLDatabase();
UserService service = new UserService(mysql);

// Easy to switch
Database postgres = new PostgreSQLDatabase();
UserService service2 = new UserService(postgres);
```

---

## Design Patterns

### Creational Patterns

#### 1. Singleton Pattern

**Ensures only one instance of a class exists**

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;
    private Connection connection;
    
    // Private constructor
    private DatabaseConnection() {
        try {
            connection = DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/mydb", "user", "password"
            );
        } catch (SQLException e) {
            throw new RuntimeException("Failed to create connection", e);
        }
    }
    
    // Thread-safe singleton
    public static synchronized DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
    
    public Connection getConnection() {
        return connection;
    }
}

// Better: Double-checked locking
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;
    private Connection connection;
    
    private DatabaseConnection() {
        // initialization
    }
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}

// Best: Enum Singleton (thread-safe by default)
public enum DatabaseConnection {
    INSTANCE;
    
    private Connection connection;
    
    DatabaseConnection() {
        // initialization
    }
    
    public Connection getConnection() {
        return connection;
    }
}

// Usage
DatabaseConnection.INSTANCE.getConnection();
```

#### 2. Factory Pattern

**Creates objects without specifying exact class**

```java
// Product interface
public interface Vehicle {
    void drive();
}

// Concrete products
public class Car implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Driving a car");
    }
}

public class Bike implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Riding a bike");
    }
}

public class Truck implements Vehicle {
    @Override
    public void drive() {
        System.out.println("Driving a truck");
    }
}

// Factory
public class VehicleFactory {
    public static Vehicle createVehicle(String type) {
        switch (type.toLowerCase()) {
            case "car":
                return new Car();
            case "bike":
                return new Bike();
            case "truck":
                return new Truck();
            default:
                throw new IllegalArgumentException("Unknown vehicle type");
        }
    }
}

// Usage
Vehicle car = VehicleFactory.createVehicle("car");
car.drive();
```

#### 3. Builder Pattern

**Constructs complex objects step by step**

```java
public class User {
    // Required parameters
    private final String firstName;
    private final String lastName;
    
    // Optional parameters
    private final String email;
    private final String phone;
    private final String address;
    private final int age;
    
    private User(UserBuilder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.email = builder.email;
        this.phone = builder.phone;
        this.address = builder.address;
        this.age = builder.age;
    }
    
    // Builder class
    public static class UserBuilder {
        // Required parameters
        private final String firstName;
        private final String lastName;
        
        // Optional parameters - initialized to default values
        private String email = "";
        private String phone = "";
        private String address = "";
        private int age = 0;
        
        public UserBuilder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        public UserBuilder email(String email) {
            this.email = email;
            return this;
        }
        
        public UserBuilder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public UserBuilder address(String address) {
            this.address = address;
            return this;
        }
        
        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
    
    @Override
    public String toString() {
        return "User{" +
            "firstName='" + firstName + '\'' +
            ", lastName='" + lastName + '\'' +
            ", email='" + email + '\'' +
            ", phone='" + phone + '\'' +
            ", address='" + address + '\'' +
            ", age=" + age +
            '}';
    }
}

// Usage
User user = new User.UserBuilder("John", "Doe")
    .email("john.doe@example.com")
    .phone("123-456-7890")
    .age(30)
    .build();
```

### Structural Patterns

#### 4. Adapter Pattern

**Allows incompatible interfaces to work together**

```java
// Existing interface
public interface MediaPlayer {
    void play(String audioType, String fileName);
}

// Advanced media player with different interface
public interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

// Concrete implementations
public class VlcPlayer implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        System.out.println("Playing vlc file: " + fileName);
    }
    
    @Override
    public void playMp4(String fileName) {
        // Do nothing
    }
}

public class Mp4Player implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        // Do nothing
    }
    
    @Override
    public void playMp4(String fileName) {
        System.out.println("Playing mp4 file: " + fileName);
    }
}

// Adapter
public class MediaAdapter implements MediaPlayer {
    private AdvancedMediaPlayer advancedPlayer;
    
    public MediaAdapter(String audioType) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedPlayer = new VlcPlayer();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedPlayer = new Mp4Player();
        }
    }
    
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedPlayer.playVlc(fileName);
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedPlayer.playMp4(fileName);
        }
    }
}

// Audio player using adapter
public class AudioPlayer implements MediaPlayer {
    private MediaAdapter mediaAdapter;
    
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("Playing mp3 file: " + fileName);
        } else if (audioType.equalsIgnoreCase("vlc") || 
                   audioType.equalsIgnoreCase("mp4")) {
            mediaAdapter = new MediaAdapter(audioType);
            mediaAdapter.play(audioType, fileName);
        } else {
            System.out.println("Invalid media type: " + audioType);
        }
    }
}

// Usage
AudioPlayer player = new AudioPlayer();
player.play("mp3", "song.mp3");
player.play("mp4", "video.mp4");
player.play("vlc", "movie.vlc");
```

#### 5. Decorator Pattern

**Adds new functionality to objects dynamically**

```java
// Component interface
public interface Coffee {
    double getCost();
    String getDescription();
}

// Concrete component
public class SimpleCoffee implements Coffee {
    @Override
    public double getCost() {
        return 2.0;
    }
    
    @Override
    public String getDescription() {
        return "Simple Coffee";
    }
}

// Decorator base class
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public double getCost() {
        return coffee.getCost();
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription();
    }
}

// Concrete decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.5;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Milk";
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.2;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Sugar";
    }
}

public class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.7;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Whip";
    }
}

// Usage
Coffee coffee = new SimpleCoffee();
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

coffee = new WhipDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());

// Output:
// Simple Coffee $2.0
// Simple Coffee, Milk $2.5
// Simple Coffee, Milk, Sugar $2.7
// Simple Coffee, Milk, Sugar, Whip $3.4
```

### Behavioral Patterns

#### 6. Strategy Pattern

**Defines family of algorithms, makes them interchangeable**

```java
// Strategy interface
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    private String cvv;
    
    public CreditCardPayment(String cardNumber, String cvv) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Credit Card: " + cardNumber);
    }
}

public class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using PayPal: " + email);
    }
}

public class CryptoPayment implements PaymentStrategy {
    private String walletAddress;
    
    public CryptoPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Crypto Wallet: " + walletAddress);
    }
}

// Context
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    private PaymentStrategy paymentStrategy;
    
    public void addItem(Item item) {
        items.add(item);
    }
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout() {
        double total = items.stream()
            .mapToDouble(Item::getPrice)
            .sum();
        
        paymentStrategy.pay(total);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.addItem(new Item("Book", 20.0));
cart.addItem(new Item("Pen", 5.0));

// Pay with credit card
cart.setPaymentStrategy(new CreditCardPayment("1234-5678-9012-3456", "123"));
cart.checkout();

// Pay with PayPal
cart.setPaymentStrategy(new PayPalPayment("user@example.com"));
cart.checkout();
```

#### 7. Observer Pattern

**Defines one-to-many dependency, notify all dependents**

```java
// Observer interface
public interface Observer {
    void update(String message);
}

// Subject interface
public interface Subject {
    void attach(Observer observer);
    void detach(Observer observer);
    void notifyObservers();
}

// Concrete subject
public class NewsAgency implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private String news;
    
    @Override
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void detach(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
    
    public void setNews(String news) {
        this.news = news;
        notifyObservers();
    }
}

// Concrete observers
public class EmailSubscriber implements Observer {
    private String email;
    
    public EmailSubscriber(String email) {
        this.email = email;
    }
    
    @Override
    public void update(String message) {
        System.out.println("Email to " + email + ": " + message);
    }
}

public class SMSSubscriber implements Observer {
    private String phone;
    
    public SMSSubscriber(String phone) {
        this.phone = phone;
    }
    
    @Override
    public void update(String message) {
        System.out.println("SMS to " + phone + ": " + message);
    }
}

public class PushNotificationSubscriber implements Observer {
    private String deviceId;
    
    public PushNotificationSubscriber(String deviceId) {
        this.deviceId = deviceId;
    }
    
    @Override
    public void update(String message) {
        System.out.println("Push notification to " + deviceId + ": " + message);
    }
}

// Usage
NewsAgency agency = new NewsAgency();

Observer emailSub = new EmailSubscriber("user@example.com");
Observer smsSub = new SMSSubscriber("123-456-7890");
Observer pushSub = new PushNotificationSubscriber("device123");

agency.attach(emailSub);
agency.attach(smsSub);
agency.attach(pushSub);

agency.setNews("Breaking News: Design Patterns are awesome!");
// All subscribers get notified

agency.detach(smsSub);
agency.setNews("Another update");
// Only email and push notification subscribers get notified
```

#### 8. Command Pattern

**Encapsulates requests as objects**

```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class Light {
    private boolean isOn = false;
    
    public void turnOn() {
        isOn = true;
        System.out.println("Light is ON");
    }
    
    public void turnOff() {
        isOn = false;
        System.out.println("Light is OFF");
    }
}

// Concrete commands
public class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
    
    @Override
    public void undo() {
        light.turnOff();
    }
}

public class LightOffCommand implements Command {
    private Light light;
    
    public LightOffCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOff();
    }
    
    @Override
    public void undo() {
        light.turnOn();
    }
}

// Invoker
public class RemoteControl {
    private Command command;
    private Stack<Command> history = new Stack<>();
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
        history.push(command);
    }
    
    public void pressUndo() {
        if (!history.isEmpty()) {
            Command lastCommand = history.pop();
            lastCommand.undo();
        }
    }
}

// Usage
Light livingRoomLight = new Light();

Command lightOn = new LightOnCommand(livingRoomLight);
Command lightOff = new LightOffCommand(livingRoomLight);

RemoteControl remote = new RemoteControl();

remote.setCommand(lightOn);
remote.pressButton();  // Light is ON

remote.setCommand(lightOff);
remote.pressButton();  // Light is OFF

remote.pressUndo();    // Light is ON (undo last command)
```

---

## Real-World LLD Examples

### 1. URL Shortener

**Design a system like bit.ly**

```java
// Entity
public class URL {
    private String shortUrl;
    private String longUrl;
    private LocalDateTime createdAt;
    private LocalDateTime expiresAt;
    private int clickCount;
    
    public URL(String shortUrl, String longUrl, LocalDateTime expiresAt) {
        this.shortUrl = shortUrl;
        this.longUrl = longUrl;
        this.createdAt = LocalDateTime.now();
        this.expiresAt = expiresAt;
        this.clickCount = 0;
    }
    
    public void incrementClickCount() {
        this.clickCount++;
    }
    
    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }
    
    // Getters and setters
}

// URL encoding service
public class Base62Encoder {
    private static final String BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    
    public String encode(long id) {
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(BASE62.charAt((int)(id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
    }
    
    public long decode(String shortUrl) {
        long id = 0;
        for (char c : shortUrl.toCharArray()) {
            id = id * 62 + BASE62.indexOf(c);
        }
        return id;
    }
}

// Repository interface
public interface URLRepository {
    URL save(URL url);
    URL findByShortUrl(String shortUrl);
    URL findByLongUrl(String longUrl);
    void delete(String shortUrl);
}

// In-memory implementation
public class InMemoryURLRepository implements URLRepository {
    private Map<String, URL> storage = new ConcurrentHashMap<>();
    private Map<String, String> longToShort = new ConcurrentHashMap<>();
    
    @Override
    public URL save(URL url) {
        storage.put(url.getShortUrl(), url);
        longToShort.put(url.getLongUrl(), url.getShortUrl());
        return url;
    }
    
    @Override
    public URL findByShortUrl(String shortUrl) {
        return storage.get(shortUrl);
    }
    
    @Override
    public URL findByLongUrl(String longUrl) {
        String shortUrl = longToShort.get(longUrl);
        return shortUrl != null ? storage.get(shortUrl) : null;
    }
    
    @Override
    public void delete(String shortUrl) {
        URL url = storage.remove(shortUrl);
        if (url != null) {
            longToShort.remove(url.getLongUrl());
        }
    }
}

// URL Shortener Service
public class URLShortenerService {
    private URLRepository repository;
    private Base62Encoder encoder;
    private AtomicLong counter;
    
    public URLShortenerService(URLRepository repository) {
        this.repository = repository;
        this.encoder = new Base62Encoder();
        this.counter = new AtomicLong(1000000); // Start from 1 million
    }
    
    public String shortenURL(String longUrl, int expiryDays) {
        // Check if URL already shortened
        URL existing = repository.findByLongUrl(longUrl);
        if (existing != null && !existing.isExpired()) {
            return existing.getShortUrl();
        }
        
        // Generate short URL
        long id = counter.getAndIncrement();
        String shortUrl = encoder.encode(id);
        
        // Create and save URL
        LocalDateTime expiresAt = LocalDateTime.now().plusDays(expiryDays);
        URL url = new URL(shortUrl, longUrl, expiresAt);
        repository.save(url);
        
        return shortUrl;
    }
    
    public String expandURL(String shortUrl) {
        URL url = repository.findByShortUrl(shortUrl);
        
        if (url == null) {
            throw new NotFoundException("Short URL not found");
        }
        
        if (url.isExpired()) {
            repository.delete(shortUrl);
            throw new ExpiredException("Short URL has expired");
        }
        
        url.incrementClickCount();
        repository.save(url);
        
        return url.getLongUrl();
    }
    
    public URLStats getStats(String shortUrl) {
        URL url = repository.findByShortUrl(shortUrl);
        if (url == null) {
            throw new NotFoundException("Short URL not found");
        }
        
        return new URLStats(
            url.getShortUrl(),
            url.getLongUrl(),
            url.getClickCount(),
            url.getCreatedAt(),
            url.getExpiresAt()
        );
    }
}

// Controller
public class URLShortenerController {
    private URLShortenerService service;
    
    public URLShortenerController(URLShortenerService service) {
        this.service = service;
    }
    
    public Response createShortURL(ShortenRequest request) {
        try {
            String shortUrl = service.shortenURL(
                request.getLongUrl(),
                request.getExpiryDays()
            );
            return Response.ok(new ShortenResponse(shortUrl));
        } catch (Exception e) {
            return Response.error(e.getMessage());
        }
    }
    
    public Response redirect(String shortUrl) {
        try {
            String longUrl = service.expandURL(shortUrl);
            return Response.redirect(longUrl);
        } catch (Exception e) {
            return Response.error(e.getMessage());
        }
    }
}

// Usage
URLRepository repository = new InMemoryURLRepository();
URLShortenerService service = new URLShortenerService(repository);
URLShortenerController controller = new URLShortenerController(service);

// Shorten URL
String shortUrl = service.shortenURL("https://www.example.com/very/long/url", 30);
System.out.println("Short URL: " + shortUrl);

// Expand URL
String longUrl = service.expandURL(shortUrl);
System.out.println("Long URL: " + longUrl);

// Get stats
URLStats stats = service.getStats(shortUrl);
System.out.println("Clicks: " + stats.getClickCount());
```

### 2. Notification Service

**Multi-channel notification system**

```java
// Notification interface
public interface Notification {
    String getRecipient();
    String getMessage();
    NotificationType getType();
}

public enum NotificationType {
    EMAIL, SMS, PUSH, IN_APP
}

// Concrete notification types
public class EmailNotification implements Notification {
    private String email;
    private String subject;
    private String body;
    
    public EmailNotification(String email, String subject, String body) {
        this.email = email;
        this.subject = subject;
        this.body = body;
    }
    
    @Override
    public String getRecipient() { return email; }
    
    @Override
    public String getMessage() { return subject + "\n" + body; }
    
    @Override
    public NotificationType getType() { return NotificationType.EMAIL; }
    
    public String getSubject() { return subject; }
    public String getBody() { return body; }
}

public class SMSNotification implements Notification {
    private String phoneNumber;
    private String message;
    
    public SMSNotification(String phoneNumber, String message) {
        this.phoneNumber = phoneNumber;
        this.message = message;
    }
    
    @Override
    public String getRecipient() { return phoneNumber; }
    
    @Override
    public String getMessage() { return message; }
    
    @Override
    public NotificationType getType() { return NotificationType.SMS; }
}

// Notification sender interface (Strategy Pattern)
public interface NotificationSender {
    boolean canHandle(NotificationType type);
    void send(Notification notification);
}

// Concrete senders
public class EmailSender implements NotificationSender {
    @Override
    public boolean canHandle(NotificationType type) {
        return type == NotificationType.EMAIL;
    }
    
    @Override
    public void send(Notification notification) {
        EmailNotification email = (EmailNotification) notification;
        System.out.println("Sending email to: " + email.getRecipient());
        System.out.println("Subject: " + email.getSubject());
        System.out.println("Body: " + email.getBody());
        // Actual email sending logic (SMTP, SendGrid, etc.)
    }
}

public class SMSSender implements NotificationSender {
    @Override
    public boolean canHandle(NotificationType type) {
        return type == NotificationType.SMS;
    }
    
    @Override
    public void send(Notification notification) {
        System.out.println("Sending SMS to: " + notification.getRecipient());
        System.out.println("Message: " + notification.getMessage());
        // Actual SMS sending logic (Twilio, AWS SNS, etc.)
    }
}

// Notification service (Chain of Responsibility + Strategy)
public class NotificationService {
    private List<NotificationSender> senders = new ArrayList<>();
    private NotificationQueue queue;
    
    public NotificationService(NotificationQueue queue) {
        this.queue = queue;
        registerDefaultSenders();
    }
    
    private void registerDefaultSenders() {
        senders.add(new EmailSender());
        senders.add(new SMSSender());
        senders.add(new PushNotificationSender());
    }
    
    public void registerSender(NotificationSender sender) {
        senders.add(sender);
    }
    
    public void sendNotification(Notification notification) {
        // Add to queue for async processing
        queue.enqueue(notification);
    }
    
    public void processNotification(Notification notification) {
        for (NotificationSender sender : senders) {
            if (sender.canHandle(notification.getType())) {
                try {
                    sender.send(notification);
                    logSuccess(notification);
                } catch (Exception e) {
                    logFailure(notification, e);
                }
                return;
            }
        }
        throw new UnsupportedOperationException(
            "No sender available for type: " + notification.getType()
        );
    }
    
    private void logSuccess(Notification notification) {
        System.out.println("Notification sent successfully: " + notification.getType());
    }
    
    private void logFailure(Notification notification, Exception e) {
        System.err.println("Failed to send notification: " + e.getMessage());
        // Retry logic or dead letter queue
    }
}

// Notification queue (for async processing)
public class NotificationQueue {
    private Queue<Notification> queue = new ConcurrentLinkedQueue<>();
    
    public void enqueue(Notification notification) {
        queue.offer(notification);
    }
    
    public Notification dequeue() {
        return queue.poll();
    }
    
    public boolean isEmpty() {
        return queue.isEmpty();
    }
}

// Notification worker (processes queue)
public class NotificationWorker implements Runnable {
    private NotificationService service;
    private NotificationQueue queue;
    private boolean running = true;
    
    public NotificationWorker(NotificationService service, NotificationQueue queue) {
        this.service = service;
        this.queue = queue;
    }
    
    @Override
    public void run() {
        while (running) {
            if (!queue.isEmpty()) {
                Notification notification = queue.dequeue();
                if (notification != null) {
                    service.processNotification(notification);
                }
            } else {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }
    
    public void stop() {
        running = false;
    }
}

// Usage
NotificationQueue queue = new NotificationQueue();
NotificationService service = new NotificationService(queue);

// Start worker thread
NotificationWorker worker = new NotificationWorker(service, queue);
Thread workerThread = new Thread(worker);
workerThread.start();

// Send notifications
Notification email = new EmailNotification(
    "user@example.com",
    "Welcome!",
    "Thanks for signing up"
);
service.sendNotification(email);

Notification sms = new SMSNotification(
    "123-456-7890",
    "Your verification code is: 123456"
);
service.sendNotification(sms);
```

This LLD guide provides a comprehensive foundation for object-oriented design, SOLID principles, design patterns, and real-world implementations. Each example demonstrates best practices and can be adapted for specific use cases.
