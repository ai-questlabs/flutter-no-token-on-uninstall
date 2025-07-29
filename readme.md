
### The Root Cause: App Data is Wiped on Uninstall

When a user uninstalls an Android app, the operating system deletes **all** of its local data. This includes:
*   **SharedPreferences:** The most common place for storing simple key-value data.
*   **Local Databases:** SQLite, Hive, Isar, etc.
*   **Files:** Any files saved in the app's private directory.

Your authentication token (like a JWT, OAuth token, or session ID) is the key that proves to your backend server who the user is. You almost certainly store this token locally after the user logs in for the first time.

**Here's the sequence of events:**

1.  **First Install:**
    *   User opens the app, sees the login screen.
    *   User authenticates (e.g., with email/password).
    *   Your backend server returns an **auth token**.
    *   Your Flutter app **saves this token** to the device's local storage (e.g., `SharedPreferences` or `flutter_secure_storage`).
    *   For all subsequent API calls (fetching products, orders), your app attaches this token to the request headers. The server validates the token and returns the user's data. **Everything works.**

2.  **Uninstall:**
    *   The user uninstalls the app.
    *   Android **deletes the saved auth token** along with all other app data. The token is now gone forever.

3.  **Reinstall:**
    *   The user reinstalls the app. The app is now in a fresh, clean state.
    *   **This is the critical part:** Your app's logic likely has a check that incorrectly assumes the user is still logged in. For example, if you are using Firebase Auth, the Firebase SDK might still have a sense of the user on the device level, but your *custom backend token* is gone.
    *   When your app tries to fetch user data (products, orders), it tries to find the auth token in local storage.
    *   It finds **nothing** (or an empty string).
    *   It makes the API call **without a valid auth token**.
    *   Your server receives the request, sees no valid token, and correctly rejects it, likely returning a `401 Unauthorized` or `403 Forbidden` error. From the user's perspective, it just looks like "no data is loading."

### How to Diagnose and Fix It

You need to implement a robust authentication flow that handles the "no token" case gracefully.

#### Step 1: Use Secure Storage for Your Token

First, you should not be storing sensitive data like auth tokens in `SharedPreferences` as it's not encrypted. The best practice is to use `flutter_secure_storage`.

1.  Add the dependency to your `pubspec.yaml`:
    ```yaml
    dependencies:
      flutter_secure_storage: ^9.0.0 # Check for the latest version
    ```

2.  Create a service or helper class to manage the token.

    ```dart
    import 'package:flutter_secure_storage/flutter_secure_storage.dart';

    class AuthStorage {
      final _storage = const FlutterSecureStorage();
      static const _tokenKey = 'auth_token';

      Future<void> saveToken(String token) async {
        await _storage.write(key: _tokenKey, value: token);
      }

      Future<String?> getToken() async {
        return await _storage.read(key: _tokenKey);
      }

      Future<void> deleteToken() async {
        await _storage.delete(key: _tokenKey);
      }
    }
    ```

#### Step 2: Check for the Token on App Startup

Your app needs to check for the existence and validity of the token every time it starts. A splash screen is a great place to handle this logic.

In your `main.dart` or a dedicated splash screen widget:

```dart
// In your Splash Screen's initState or a similar startup logic location

@override
void initState() {
  super.initState();
  _checkAuthentication();
}

Future<void> _checkAuthentication() async {
  final authStorage = AuthStorage();
  final token = await authStorage.getToken();

  // Delay for a moment to show the splash screen
  await Future.delayed(const Duration(seconds: 2));

  if (token != null && token.isNotEmpty) {
    // Optional but recommended: Validate the token with the server.
    // An expired token is as bad as no token.
    // bool isTokenValid = await ApiService.validateToken(token);
    // if (isTokenValid) {
    //   Navigator.of(context).pushReplacementNamed('/home');
    // } else {
    //   Navigator.of(context).pushReplacementNamed('/login');
    // }

    // Simple check (without server validation):
    print("Token found. Navigating to home.");
    Navigator.of(context).pushReplacementNamed('/home');
  } else {
    // No token found, user must log in.
    print("No token found. Navigating to login.");
    Navigator.of(context).pushReplacementNamed('/login');
  }
}
```

#### Step 3: Attach the Token to Every API Request

You need to ensure that every authenticated API call includes the token. Using an interceptor with a networking package like `dio` is the cleanest way to do this.

1.  Add `dio` to your `pubspec.yaml`:
    ```yaml
    dependencies:
      dio: ^5.3.3 # Check for the latest version
    ```

2.  Set up your Dio instance with an interceptor.

    ```dart
    import 'package:dio/dio.dart';

    class ApiClient {
      final Dio _dio;
      final AuthStorage _authStorage = AuthStorage();

      ApiClient() : _dio = Dio(BaseOptions(baseUrl: 'https://your.api.url/')) {
        _dio.interceptors.add(
          InterceptorsWrapper(
            onRequest: (options, handler) async {
              // Get the token from secure storage
              final token = await _authStorage.getToken();
              if (token != null) {
                // Add the token to the Authorization header
                options.headers['Authorization'] = 'Bearer $token';
              }
              // Continue with the request
              return handler.next(options);
            },
            onError: (DioException e, handler) async {
              // If we get a 401, the token is likely expired or invalid.
              // You can implement token refresh logic here or simply
              // log the user out.
              if (e.response?.statusCode == 401) {
                print("Auth token is invalid. Logging out.");
                await _authStorage.deleteToken();
                // Here you should navigate the user to the login screen.
                // This is best handled with a global state manager.
              }
              return handler.next(e);
            },
          ),
        );
      }

      // Your method to fetch products
      Future<Response> getProducts() {
        return _dio.get('/products');
      }

      // Your method to fetch orders
      Future<Response> getOrders() {
        return _dio.get('/orders');
      }
    }
    ```

### Summary of the Correct Flow

1.  **Login:** User authenticates -> Server returns token -> App saves token using `flutter_secure_storage`.
2.  **App Start (First time or Reinstall):**
    *   App checks `flutter_secure_storage` for a token.
    *   **If token exists:** Navigate to the home screen.
    *   **If token does NOT exist:** Navigate to the login screen.
3.  **API Calls:** The `Dio` interceptor automatically attaches the saved token to every outgoing request.
4.  **Logout:** When the user logs out, you must explicitly call `authStorage.deleteToken()` to remove the token from storage.

By implementing this robust flow, your app will correctly handle being reinstalled because it will always check for the necessary token before trying to fetch protected data. If the token is missing, it will correctly force the user to log in again.
