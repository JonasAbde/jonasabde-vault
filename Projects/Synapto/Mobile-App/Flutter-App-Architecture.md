# Flutter Mobile App Architecture

**Complete guide to the Synapto Flutter mobile app for field workers**

---

## Overview

The Synapto Flutter app is an **offline-first mobile application** for field workers to:
- View and respond to customer inquiries
- Manage customer information
- Handle bookings and calendar
- Chat with AI assistant
- Operate without internet connection (syncs when online)

**Platforms**: Android, iOS, Windows (development)

---

## Architecture Pattern

### Feature-Based Structure

```
lib/
├── config/              # Environment configuration
│   ├── dev.dart
│   ├── staging.dart
│   └── production.dart
├── core/               # Shared infrastructure
│   ├── api/            # API client (Retrofit + Dio)
│   ├── models/         # Shared data models
│   ├── providers/      # Global Riverpod providers
│   ├── router/         # Navigation routing
│   ├── services/       # Secure storage, mutation queue
│   ├── theme/          # Material design theme
│   └── utils/          # Utility functions
├── features/           # Feature modules
│   ├── auth/
│   │   ├── models/
│   │   ├── providers/
│   │   ├── screens/
│   │   └── services/
│   ├── dashboard/
│   ├── inquiries/
│   ├── customers/
│   ├── bookings/
│   ├── calendar/
│   ├── chat/
│   ├── analytics/
│   └── settings/
└── shared/             # Shared UI widgets
    ├── main_scaffold.dart
    ├── empty_state.dart
    └── error_view.dart
```

**Pattern**: Each feature is self-contained with its own models, providers, screens, and services.

---

## State Management

### Riverpod + StateNotifier

**Why Riverpod?**
- Type-safe providers
- Compile-time safety
- Testable
- No BuildContext required

### Global Providers

**Location**: `lib/core/providers/app_providers.dart`

```dart
// Secure storage for sensitive data
final secureStorageProvider = Provider((ref) => FlutterSecureStorage());

// Shared preferences for app settings
final sharedPreferencesProvider = FutureProvider((ref) async {
  return await SharedPreferences.getInstance();
});

// Dio HTTP client
final dioProvider = Provider((ref) {
  final dio = Dio(BaseOptions(
    baseUrl: Environment.apiUrl,
    connectTimeout: Duration(seconds: 30),
    receiveTimeout: Duration(seconds: 30),
  ));
  
  // Add interceptors
  dio.interceptors.add(LogInterceptor());
  dio.interceptors.add(AuthInterceptor());
  
  return dio;
});

// API client (Retrofit)
final apiClientProvider = Provider((ref) {
  final dio = ref.watch(dioProvider);
  final tenantId = ref.watch(currentTenantIdProvider);
  return ApiClient(dio, baseUrl: Environment.apiUrl, tenantId: tenantId);
});

// Mutation queue service (offline support)
final mutationQueueProvider = Provider((ref) {
  final apiClient = ref.watch(apiClientProvider);
  return MutationQueueService(apiClient);
});
```

### Feature Providers

**Example**: Inquiry List Provider

**Location**: `lib/features/inquiries/providers/inquiry_list_provider.dart`

```dart
// State class
class InquiryListState {
  final List<Inquiry> inquiries;
  final bool isLoading;
  final String? error;
  final InquiryFilter filter;
  final String searchQuery;
  
  InquiryListState({
    this.inquiries = const [],
    this.isLoading = false,
    this.error,
    this.filter = InquiryFilter.all,
    this.searchQuery = '',
  });
  
  InquiryListState copyWith({
    List<Inquiry>? inquiries,
    bool? isLoading,
    String? error,
    InquiryFilter? filter,
    String? searchQuery,
  }) {
    return InquiryListState(
      inquiries: inquiries ?? this.inquiries,
      isLoading: isLoading ?? this.isLoading,
      error: error ?? this.error,
      filter: filter ?? this.filter,
      searchQuery: searchQuery ?? this.searchQuery,
    );
  }
}

// StateNotifier
class InquiryListNotifier extends StateNotifier<InquiryListState> {
  final ApiClient _apiClient;
  final Box<Inquiry> _inquiryBox;
  
  InquiryListNotifier(this._apiClient, this._inquiryBox)
      : super(InquiryListState());
  
  Future<void> fetchInquiries() async {
    state = state.copyWith(isLoading: true, error: null);
    
    try {
      // Try API call
      final inquiries = await _apiClient.getInquiries();
      
      // Save to Hive cache
      await _inquiryBox.clear();
      await _inquiryBox.addAll(inquiries);
      
      state = state.copyWith(
        inquiries: inquiries,
        isLoading: false,
      );
    } catch (e) {
      // Fallback to cached data
      final cachedInquiries = _inquiryBox.values.toList();
      
      state = state.copyWith(
        inquiries: cachedInquiries,
        isLoading: false,
        error: e.toString(),
      );
    }
  }
  
  void setFilter(InquiryFilter filter) {
    state = state.copyWith(filter: filter);
  }
  
  void setSearchQuery(String query) {
    state = state.copyWith(searchQuery: query);
  }
}

// Provider
final inquiryListProvider = StateNotifierProvider<InquiryListNotifier, InquiryListState>((ref) {
  final apiClient = ref.watch(apiClientProvider);
  final inquiryBox = Hive.box<Inquiry>('inquiries');
  return InquiryListNotifier(apiClient, inquiryBox);
});
```

### Using Providers in Widgets

```dart
class InquiryListScreen extends ConsumerStatefulWidget {
  @override
  ConsumerState<InquiryListScreen> createState() => _InquiryListScreenState();
}

class _InquiryListScreenState extends ConsumerState<InquiryListScreen> {
  @override
  void initState() {
    super.initState();
    // Fetch data on init
    Future.microtask(() => 
      ref.read(inquiryListProvider.notifier).fetchInquiries()
    );
  }
  
  @override
  Widget build(BuildContext context) {
    // Watch provider for changes
    final state = ref.watch(inquiryListProvider);
    
    if (state.isLoading) {
      return LoadingIndicator();
    }
    
    if (state.error != null) {
      return ErrorView(error: state.error!);
    }
    
    return ListView.builder(
      itemCount: state.inquiries.length,
      itemBuilder: (context, index) {
        final inquiry = state.inquiries[index];
        return InquiryCard(inquiry: inquiry);
      },
    );
  }
}
```

---

## Offline-First Implementation

### Mutation Queue Service

**Location**: `lib/core/services/mutation_queue_service.dart`

**Purpose**: Store API mutations when offline, sync when online.

```dart
class MutationQueueService {
  final ApiClient _apiClient;
  final Box<QueuedMutation> _mutationBox;
  StreamSubscription? _connectivitySubscription;
  
  MutationQueueService(this._apiClient) 
      : _mutationBox = Hive.box<QueuedMutation>('mutation_queue') {
    // Listen to connectivity changes
    _connectivitySubscription = Connectivity()
        .onConnectivityChanged
        .listen((results) {
      if (results.any((r) => r != ConnectivityResult.none)) {
        processQueue();
      }
    });
  }
  
  /// Queue a mutation for later execution
  Future<void> queueMutation(
    String type,
    String entityId,
    Map<String, dynamic> data,
  ) async {
    final mutation = QueuedMutation(
      id: uuid.v4(),
      type: type,
      entityId: entityId,
      data: data,
      timestamp: DateTime.now(),
      attempts: 0,
    );
    
    await _mutationBox.add(mutation);
    
    // Try to process immediately if online
    if (await _isOnline()) {
      await processQueue();
    }
  }
  
  /// Process all queued mutations
  Future<void> processQueue() async {
    if (!await _isOnline()) return;
    
    final mutations = _mutationBox.values.toList()
      ..sort((a, b) => a.timestamp.compareTo(b.timestamp));
    
    for (final mutation in mutations) {
      try {
        await _executeMutation(mutation);
        await _mutationBox.delete(mutation.key);
      } catch (e) {
        // Increment attempts
        mutation.attempts++;
        await mutation.save();
        
        // Give up after 5 attempts
        if (mutation.attempts >= 5) {
          await _mutationBox.delete(mutation.key);
        }
      }
    }
  }
  
  Future<void> _executeMutation(QueuedMutation mutation) async {
    switch (mutation.type) {
      case 'updateResponse':
        await _apiClient.updateInquiryResponse(
          mutation.entityId,
          mutation.data['response'],
        );
        break;
      case 'sendResponse':
        await _apiClient.sendInquiryResponse(mutation.entityId);
        break;
      case 'archiveInquiry':
        await _apiClient.archiveInquiry(mutation.entityId);
        break;
      default:
        throw Exception('Unknown mutation type: ${mutation.type}');
    }
  }
  
  Future<bool> _isOnline() async {
    final results = await Connectivity().checkConnectivity();
    return results.any((r) => r != ConnectivityResult.none);
  }
}
```

### Mutation Types

| Type | Description | Data |
|------|-------------|------|
| `updateResponse` | Update inquiry response | `{response: "..."}` |
| `sendResponse` | Send response email | `{}` |
| `archiveInquiry` | Archive inquiry | `{}` |
| `updateCustomer` | Update customer info | `{name: "...", email: "..."}` |
| `createBooking` | Create new booking | `{date: "...", hours: 3}` |

### Local Caching with Hive

**Hive boxes** store data locally for offline access:

```dart
// Initialize Hive boxes
await Hive.initFlutter();

// Register adapters (generated code)
Hive.registerAdapter(InquiryAdapter());
Hive.registerAdapter(CustomerAdapter());
Hive.registerAdapter(BookingAdapter());
Hive.registerAdapter(QueuedMutationAdapter());

// Open boxes
await Hive.openBox<Inquiry>('inquiries');
await Hive.openBox<Customer>('customers');
await Hive.openBox<Booking>('bookings');
await Hive.openBox<QueuedMutation>('mutation_queue');
```

**Cache Strategy**:
1. Load cached data immediately (instant UI)
2. Fetch fresh data from API
3. Update cache and UI with new data
4. If API fails, use cached data

```dart
Future<List<Inquiry>> getInquiries() async {
  final box = Hive.box<Inquiry>('inquiries');
  
  // Return cached data immediately
  final cachedData = box.values.toList();
  
  try {
    // Fetch fresh data
    final freshData = await _apiClient.getInquiries();
    
    // Update cache
    await box.clear();
    await box.addAll(freshData);
    
    return freshData;
  } catch (e) {
    // Use cached data if API fails
    return cachedData;
  }
}
```

---

## API Client

### Retrofit + Dio

**Location**: `lib/core/api/api_client.dart`

**Generated code** from annotations:

```dart
@RestApi(baseUrl: "/api")
abstract class ApiClient {
  factory ApiClient(Dio dio, {String baseUrl, String tenantId}) = _ApiClient;
  
  // Inquiries
  @GET("/inquiries")
  Future<List<Inquiry>> getInquiries(@Header("X-Tenant-ID") String tenantId);
  
  @GET("/inquiries/{id}")
  Future<Inquiry> getInquiry(
    @Path("id") String id,
    @Header("X-Tenant-ID") String tenantId,
  );
  
  @PUT("/inquiries/{id}/response")
  Future<void> updateInquiryResponse(
    @Path("id") String id,
    @Body() String response,
  );
  
  @POST("/inquiries/{id}/send")
  Future<void> sendInquiryResponse(@Path("id") String id);
  
  // Customers
  @GET("/customers")
  Future<List<Customer>> getCustomers();
  
  @POST("/customers")
  Future<Customer> createCustomer(@Body() Customer customer);
  
  // Bookings
  @GET("/bookings")
  Future<List<Booking>> getBookings(
    @Query("start_date") String startDate,
    @Query("end_date") String endDate,
  );
  
  @POST("/bookings")
  Future<Booking> createBooking(@Body() Booking booking);
}
```

**Build command**: `flutter pub run build_runner build`

### Authentication Interceptor

```dart
class AuthInterceptor extends Interceptor {
  final FlutterSecureStorage _storage;
  
  AuthInterceptor(this._storage);
  
  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    // Add API token from secure storage
    final token = await _storage.read(key: 'api_token');
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    
    // Add tenant ID
    final tenantId = await _storage.read(key: 'tenant_id');
    if (tenantId != null) {
      options.headers['X-Tenant-ID'] = tenantId;
    }
    
    handler.next(options);
  }
  
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // Redirect to login on 401
    if (err.response?.statusCode == 401) {
      // Clear credentials
      _storage.deleteAll();
      // Navigate to login (via navigation service)
    }
    
    handler.next(err);
  }
}
```

---

## Secure Storage

### FlutterSecureStorage

**Stores encrypted data**:
- API token
- Tenant ID
- User email
- Refresh token

```dart
final storage = FlutterSecureStorage();

// Save
await storage.write(key: 'api_token', value: token);
await storage.write(key: 'tenant_id', value: tenantId);

// Read
final token = await storage.read(key: 'api_token');

// Delete
await storage.delete(key: 'api_token');
await storage.deleteAll();
```

**Platform-specific encryption**:
- **Android**: Keystore
- **iOS**: Keychain
- **Windows**: CredentialManager

---

## Navigation

### Router Configuration

**Location**: `lib/core/router/app_router.dart`

```dart
class AppRouter {
  static const String splash = '/';
  static const String login = '/login';
  static const String dashboard = '/dashboard';
  static const String inquiries = '/inquiries';
  static const String inquiryDetails = '/inquiries/:id';
  static const String customers = '/customers';
  static const String calendar = '/calendar';
  
  static Route<dynamic> generateRoute(RouteSettings settings) {
    switch (settings.name) {
      case splash:
        return MaterialPageRoute(builder: (_) => SplashScreen());
      case login:
        return MaterialPageRoute(builder: (_) => LoginScreen());
      case dashboard:
        return MaterialPageRoute(builder: (_) => DashboardScreen());
      case inquiryDetails:
        final id = settings.arguments as String;
        return MaterialPageRoute(
          builder: (_) => InquiryDetailsScreen(id: id),
        );
      default:
        return MaterialPageRoute(
          builder: (_) => Scaffold(
            body: Center(child: Text('Route not found: ${settings.name}')),
          ),
        );
    }
  }
}
```

### Navigation Usage

```dart
// Navigate to screen
Navigator.pushNamed(context, AppRouter.inquiries);

// Navigate with arguments
Navigator.pushNamed(
  context, 
  AppRouter.inquiryDetails,
  arguments: inquiryId,
);

// Replace current screen
Navigator.pushReplacementNamed(context, AppRouter.dashboard);

// Pop back
Navigator.pop(context);
```

---

## Testing

### Widget Testing

```dart
// test/features/inquiries/inquiry_list_screen_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  testWidgets('InquiryListScreen displays inquiries', (tester) async {
    // Create test data
    final inquiries = [
      Inquiry(id: '1', subject: 'Test 1'),
      Inquiry(id: '2', subject: 'Test 2'),
    ];
    
    // Override provider with test data
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          inquiryListProvider.overrideWith((ref) => 
            InquiryListState(inquiries: inquiries)
          ),
        ],
        child: MaterialApp(home: InquiryListScreen()),
      ),
    );
    
    // Verify UI
    expect(find.text('Test 1'), findsOneWidget);
    expect(find.text('Test 2'), findsOneWidget);
  });
}
```

### Integration Testing

```dart
// integration_test/login_flow_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  testWidgets('Login flow E2E test', (tester) async {
    await tester.pumpWidget(MyApp());
    
    // Enter API key
    await tester.enterText(
      find.byKey(Key('api_key_field')),
      'test-api-key',
    );
    
    // Tap login button
    await tester.tap(find.byKey(Key('login_button')));
    await tester.pumpAndSettle();
    
    // Verify navigation to dashboard
    expect(find.text('Dashboard'), findsOneWidget);
  });
}
```

---

## Related Documentation
- [[Mobile-App/Offline-Strategy]] - Detailed offline implementation
- [[Mobile-App/State-Management]] - Riverpod patterns
- [[API-Documentation]] - Backend API reference
- [[Testing/Mobile-Testing]] - Mobile testing guide

#flutter #mobile #offline-first #riverpod #dart
