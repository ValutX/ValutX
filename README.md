import 'package:flutter/material.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:http/http.dart' as http;
import 'package:fl_chart/fl_chart.dart';
import 'dart:convert';
import 'package:tflite_flutter/tflite_flutter.dart';

// Model: User
class User {
  final String username, email, token;
  User({required this.username, required this.email, required this.token});
  factory User.fromJson(Map<String, dynamic> json) => User(
      username: json['username'], email: json['email'], token: json['token']);
}

// API Service
class ApiService {
  static const baseUrl = 'https://api.vaultx.com';
  
  Future<User?> login(String username, String password) async {
    final response = await http.post(Uri.parse('$baseUrl/login'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'username': username, 'password': password}),
    );
    if (response.statusCode == 200) return User.fromJson(jsonDecode(response.body));
    throw Exception('Failed to login');
  }

  Future<List<dynamic>> fetchMarketData() async {
    final response = await http.get(Uri.parse('$baseUrl/market-data'));
    if (response.statusCode == 200) return jsonDecode(response.body);
    throw Exception('Failed to load market data');
  }

  Future<void> placeTrade(String token, Map<String, dynamic> tradeData) async {
    final response = await http.post(Uri.parse('$baseUrl/place-trade'),
      headers: {'Content-Type': 'application/json', 'Authorization': 'Bearer $token'},
      body: jsonEncode(tradeData),
    );
    if (response.statusCode != 200) throw Exception('Failed to place trade');
  }
}

// Secure Token Storage
class SecureStorageService {
  final storage = FlutterSecureStorage();

  Future<void> saveToken(String token) async {
    await storage.write(key: 'auth_token', value: token);
  }

  Future<String?> getToken() async {
    return await storage.read(key: 'auth_token');
  }

  Future<void> deleteToken() async {
    await storage.delete(key: 'auth_token');
  }
}

// Market Prediction using AI (TensorFlow Lite)
class MarketPredictionService {
  final Interpreter interpreter;
  MarketPredictionService() : interpreter = Interpreter.fromAsset('model.tflite');

  List<double> predict(List<double> inputData) {
    var output = List.filled(1, 0.0);
    interpreter.run(inputData, output);
    return output;
  }
}

// Market Data Visualization (Line Chart)
class MarketTrendChart extends StatelessWidget {
  final List<FlSpot> dataPoints;

  MarketTrendChart(this.dataPoints);

  @override
  Widget build(BuildContext context) {
    return LineChart(
      LineChartData(
        lineBarsData: [
          LineChartBarData(
            spots: dataPoints,
            isCurved: true,
            colors: [Colors.blue],
          ),
        ],
        titlesData: FlTitlesData(
          bottomTitles: SideTitles(showTitles: true),
          leftTitles: SideTitles(showTitles: true),
        ),
      ),
    );
  }
}

// Login Screen
class LoginScreen extends StatefulWidget {
  @override _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _usernameController = TextEditingController();
  final _passwordController = TextEditingController();
  final _apiService = ApiService();
  final _secureStorage = SecureStorageService();

  void _login() async {
    try {
      final user = await _apiService.login(_usernameController.text, _passwordController.text);
      if (user != null) {
        await _secureStorage.saveToken(user.token);
        Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => HomeScreen(user: user)));
      }
    } catch (error) { print('Login failed: $error'); }
  }

  @override Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('VaultX Login')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: <Widget>[
            TextField(controller: _usernameController, decoration: InputDecoration(labelText: 'Username')),
            TextField(controller: _passwordController, decoration: InputDecoration(labelText: 'Password'), obscureText: true),
            SizedBox(height: 20),
            ElevatedButton(onPressed: _login, child: Text('Login')),
          ],
        ),
      ),
    );
  }
}

// Home Screen with Market Data and Trading
class HomeScreen extends StatefulWidget {
  final User user;
  HomeScreen({required this.user});

  @override _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final ApiService _apiService = ApiService();
  final MarketPredictionService _predictionService = MarketPredictionService();
  List<FlSpot> _dataPoints = [];

  void _fetchMarketData() async {
    try {
      final data = await _apiService.fetchMarketData();
      setState(() {
        _dataPoints = List.generate(data.length, (index) => FlSpot(index.toDouble(), data[index]['price'].toDouble()));
      });
    } catch (error) { print('Failed to load market data: $error'); }
  }

  void _placeTrade() async {
    Map<String, dynamic> tradeData = {'symbol': 'BTCUSD', 'amount': 0.1, 'orderType': 'market'};
    try {
      await _apiService.placeTrade(widget.user.token, tradeData);
      print('Trade placed successfully');
    } catch (error) { print('Failed to place trade: $error'); }
  }

  List<double> _generatePredictionInput() {
    return _dataPoints.map((spot) => spot.y).toList();
  }

  void _predictMarket() {
    List<double> inputData = _generatePredictionInput();
    var prediction = _predictionService.predict(inputData);
    print('Market Prediction: $prediction');
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Welcome, ${widget.user.username}')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            ElevatedButton(onPressed: _fetchMarketData, child: Text('Fetch Market Data')),
            SizedBox(height: 20),
            _dataPoints.isNotEmpty ? Container(height: 300, child: MarketTrendChart(_dataPoints)) : Container(),
            SizedBox(height: 20),
            ElevatedButton(onPressed: _placeTrade, child: Text('Place Trade')),
            SizedBox(height: 20),
            ElevatedButton(onPressed: _predictMarket, child: Text('Predict Market')),
          ],
        ),
      ),
    );
  }
}

// Main Entry Point
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override Widget build(BuildContext context) {
    return MaterialApp(
      title: 'VaultX',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: LoginScreen(),
    );
  }
}
