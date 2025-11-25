# Shared Preference Service — README

This document explains the *purpose, structure, and usage* of the Shared Preference Service within a Flutter Clean Architecture project that follows **SOLID**, **BLoC**, and **Dependency Injection** principles.

---

## 📌 Overview

The Shared Preference Service provides a **local key‑value storage layer** for persisting lightweight data such as tokens, user preferences, onboarding flags, and more.

This service is implemented in a way that:

* Follows SOLID principles (Interface → Implementation)
* Supports Dependency Injection
* Keeps UI / BLoC layers independent from SharedPreferences
* Allows replacing the storage engine in the future (e.g., Hive, SecureStorage)

---

## 📁 Folder Structure

```
core/
└── services/
    ├── storage/
    │   ├── storage_service.dart
    │   └── shared_pref_service_impl.dart

core/
└── di/
    └── injector.dart
```

---

## 1️⃣ storage_service.dart (Abstract Interface)

### **Why this file exists**

* To define a **contract** for local storage.
* To ensure that the app depends on *abstractions* instead of SharedPreferences directly.
* To make the storage system easily replaceable.
* To make unit testing simpler through mocking.

### **What this file is used for**

* Declares method signatures for saving and retrieving values.
* Provides consistent behaviour across all storage implementations.
* Ensures the rest of the app interacts with a clean and predictable interface.

Methods typically include:

* Save string/bool/int
* Read string/bool/int
* Remove a key
* Clear all storage

### **Code Example**

```dart
abstract class StorageService {
  /// Save a string value
  Future<bool> saveString(String key, String value);

  /// Read a string value
  String? getString(String key);

  /// Save a boolean value
  Future<bool> saveBool(String key, bool value);

  /// Read boolean value
  bool? getBool(String key);

  /// Save integer value
  Future<bool> saveInt(String key, int value);

  /// Read integer value
  int? getInt(String key);

  /// Remove a specific key
  Future<bool> remove(String key);

  /// Clear all keys
  Future<bool> clearAll();
}
```

---

## 2️⃣ shared_pref_service_impl.dart (Concrete Implementation)

### **Why this file exists**

* To provide the actual SharedPreferences-based implementation of `StorageService`.
* To keep SharedPreferences usage isolated from the rest of the app.
* To avoid plugin calls leaking into domain or presentation layers.

### **What this file is used for**

* Initializes and interacts with SharedPreferences.
* Implements all methods defined in the abstract interface.
* Converts values into key-value storage.
* Provides a single, consistent implementation for all storage operations.

### **Code Example**

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'storage_service.dart';

class SharedPrefServiceImpl implements StorageService {
  SharedPreferences? _prefs;

  /// Initialize SharedPreferences (lazy loading)
  Future<void> _init() async {
    _prefs ??= await SharedPreferences.getInstance();
  }

  @override
  Future<bool> saveString(String key, String value) async {
    await _init();
    return _prefs!.setString(key, value);
  }

  @override
  String? getString(String key) {
    return _prefs?.getString(key);
  }

  @override
  Future<bool> saveBool(String key, bool value) async {
    await _init();
    return _prefs!.setBool(key, value);
  }

  @override
  bool? getBool(String key) {
    return _prefs?.getBool(key);
  }

  @override
  Future<bool> saveInt(String key, int value) async {
    await _init();
    return _prefs!.setInt(key, value);
  }

  @override
  int? getInt(String key) {
    return _prefs?.getInt(key);
  }

  @override
  Future<bool> remove(String key) async {
    await _init();
    return _prefs!.remove(key);
  }

  @override
  Future<bool> clearAll() async {
    await _init();
    return _prefs!.clear();
  }
}
```

---

## 3️⃣ injector.dart (Dependency Injection Registration)

### **Why this file exists**

* To register `StorageService` with its implementation so the app can request it anywhere.
* To ensure the app always uses the correct storage implementation.
* To support lazy initialization and test-friendly architecture.

### **What this file is used for**

* Registers SharedPrefServiceImpl as the default StorageService.
* Provides access to the service using DI (e.g., via GetIt).
* Keeps construction logic centralized and manageable.

### **Code Example**

```dart
import 'package:get_it/get_it.dart';
import '../services/storage/shared_pref_service_impl.dart';
import '../services/storage/storage_service.dart';

final injector = GetIt.instance;

Future<void> initializeDependencies() async {
  // Storage Service
  injector.registerLazySingleton<StorageService>(
    () => SharedPrefServiceImpl(),
  );

  // Add other services here as your app grows
  // Example:
  // injector.registerLazySingleton<ApiService>(() => ApiServiceImpl());
  // injector.registerFactory<AuthRepository>(() => AuthRepositoryImpl());
}
```

**Note:** Call `initializeDependencies()` in your `main.dart` before running the app:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initializeDependencies();
  runApp(MyApp());
}
```

---

## 4️⃣ How This Service Is Used in the App

### **Why this matters**

* Avoids direct calls to SharedPreferences inside UI or BLoC.
* Keeps code clean, testable, and replaceable.

### **What developers can do with it**

* Save login tokens
* Store onboarding completion
* Save theme mode or language
* Cache user flags

### **Usage Examples**

#### **Basic Usage**

```dart
// Get the service from dependency injection
final storage = injector.get<StorageService>();

// Save a string value
await storage.saveString('token', 'abc123');

// Read a string value
final token = storage.getString('token');
print('Token: $token');

// Save a boolean value
await storage.saveBool('isOnboardingCompleted', true);

// Read a boolean value
final isCompleted = storage.getBool('isOnboardingCompleted') ?? false;

// Save an integer value
await storage.saveInt('loginCount', 5);

// Read an integer value
final count = storage.getInt('loginCount') ?? 0;

// Remove a specific key
await storage.remove('token');

// Clear all stored data
await storage.clearAll();
```

#### **Usage in BLoC**

```dart
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final StorageService _storageService;

  AuthBloc(this._storageService) : super(AuthInitial()) {
    on<LoginEvent>(_onLogin);
    on<LogoutEvent>(_onLogout);
  }

  Future<void> _onLogin(LoginEvent event, Emitter<AuthState> emit) async {
    // Save token after successful login
    await _storageService.saveString('auth_token', event.token);
    await _storageService.saveBool('isLoggedIn', true);
    emit(AuthAuthenticated());
  }

  Future<void> _onLogout(LogoutEvent event, Emitter<AuthState> emit) async {
    // Clear stored data on logout
    await _storageService.remove('auth_token');
    await _storageService.saveBool('isLoggedIn', false);
    emit(AuthUnauthenticated());
  }
}
```

#### **Usage in Repository**

```dart
class AuthRepository {
  final StorageService _storageService;

  AuthRepository(this._storageService);

  Future<void> saveUserSession(String token) async {
    await _storageService.saveString('auth_token', token);
  }

  String? getUserToken() {
    return _storageService.getString('auth_token');
  }

  Future<void> clearSession() async {
    await _storageService.remove('auth_token');
  }
}
```

---

## ⭐ Benefits of This Architecture

### **Why this approach is recommended**

* Follows SOLID principles
* Separates concerns (UI, domain, data)
* Storage logic can be swapped anytime (Hive, secure storage)
* Easy to test using mock implementations
* Reduces coupling to Flutter plugins

---

## 🎯 Summary

The Shared Preference Service provides a clean, abstracted, test-friendly way of handling persistent key-value storage within a Flutter app. It integrates seamlessly with Clean Architecture, SOLID principles, BLoC pattern, and Dependency Injection.

If you want a **Secure Storage version**, **Hive version**, or **Encrypted storage version**, just ask!
