# piefly-app
future life
Main app entry point for Piefly Food Delivery with payment & tracking
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:geolocator/geolocator.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:url_launcher/url_launcher.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(PieflyApp());
}

class PieflyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Piefly Food Delivery',
      theme: ThemeData(primarySwatch: Colors.deepOrange),
      home: AuthScreen(),
    );
  }
}

class AuthScreen extends StatefulWidget {
  @override
  _AuthScreenState createState() => _AuthScreenState();
}

class _AuthScreenState extends State {
  final emailController = TextEditingController();
  final passwordController = TextEditingController();

  void register() async {
    try {
      await FirebaseAuth.instance.createUserWithEmailAndPassword(
        email: emailController.text,
        password: passwordController.text,
      );
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(builder: (context) => HomeScreen()),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Error: $e')));
    }
  }

  void login() async {
    try {
      await FirebaseAuth.instance.signInWithEmailAndPassword(
        email: emailController.text,
        password: passwordController.text,
      );
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(builder: (context) => HomeScreen()),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Error: $e')));
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Padding(
        padding: EdgeInsets.all(20),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text("Welcome to Piefly", style: TextStyle(fontSize: 24)),
            TextField(
              controller: emailController,
              decoration: InputDecoration(labelText: "Email"),
            ),
            TextField(
              controller: passwordController,
              obscureText: true,
              decoration: InputDecoration(labelText: "Password"),
            ),
            SizedBox(height: 20),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceEvenly,
              children: [
                ElevatedButton(onPressed: login, child: Text("Login")),
                ElevatedButton(onPressed: register, child: Text("Register")),
              ],
            ),
          ],
        ),
      ),
    );
  }
}

class HomeScreen extends StatefulWidget {
  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State {
  final List> foods = [
    {"name": "Pepperoni Pizza", "price": 8.99, "image": "https://example.com/pizza.jpg"},
    {"name": "Cheese Burger", "price": 6.50, "image": "https://example.com/burger.jpg"},
  ];

  final List> drinks = [
    {"name": "Coca Cola", "price": 1.99, "image": "https://example.com/coke.jpg"},
    {"name": "Orange Juice", "price": 2.50, "image": "https://example.com/juice.jpg"},
  ];

  List> cart = [];

  void addToCart(Map item) {
    setState(() {
      cart.add(item);
    });
  }

  void goToCheckout() {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => CheckoutScreen(cart: cart),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Piefly Menu"),
        actions: [
          IconButton(
            icon: Icon(Icons.shopping_cart),
            onPressed: goToCheckout,
          )
        ],
      ),
      body: ListView(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Text("Food", style: TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
          ),
          ...foods.map((item) => Card(
                child: ListTile(
                  leading: Image.network(item['image'], width: 50, height: 50, fit: BoxFit.cover),
                  title: Text(item['name']),
                  trailing: Text("$${item['price']}"),
                  onTap: () => addToCart(item),
                ),
              )),
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Text("Drinks", style: TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
          ),
          ...drinks.map((item) => Card(
                child: ListTile(
                  leading: Image.network(item['image'], width: 50, height: 50, fit: BoxFit.cover),
                  title: Text(item['name']),
                  trailing: Text("$${item['price']}"),
                  onTap: () => addToCart(item),
                ),
              )),
        ],
      ),
    );
  }
}

class CheckoutScreen extends StatelessWidget {
  final List> cart;

  CheckoutScreen({required this.cart});

  double get total => cart.fold(0, (sum, item) => sum + item['price']);

  void confirmOrder(BuildContext context) async {
    Position position = await Geolocator.getCurrentPosition(desiredAccuracy: LocationAccuracy.high);
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(
      content: Text('Order placed! Your location: ${position.latitude}, ${position.longitude}'),
    ));
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (_) => TrackingScreen(userLat: position.latitude, userLng: position.longitude),
      ),
    );
  }

  void launchPaymentGateway(String method) async {
    String url = "https://paymentgateway.com/pay?method=$method&amount=${total.toStringAsFixed(2)}";
    if (await canLaunch(url)) {
      await launch(url);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Checkout")),
      body: Column(
        children: [
          Expanded(
            child: ListView(
              children: cart.map((item) => ListTile(
                title: Text(item['name']),
                trailing: Text("$${item['price']}"),
              )).toList(),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(16.0),
            child: Column(
              children: [
                Text("Total: \$${total.toStringAsFixed(2)}", style: TextStyle(fontSize: 18)),
                SizedBox(height: 10),
                ElevatedButton(
                  onPressed: () => confirmOrder(context),
                  child: Text("Pay via Airtel Money"),
                ),
                ElevatedButton(
                  onPressed: () => launchPaymentGateway("mpesa"),
                  child: Text("Pay via M-Pesa"),
                ),
                ElevatedButton(
                  onPressed: () => launchPaymentGateway("paypal"),
                  child: Text("Pay via PayPal"),
                ),
                ElevatedButton(
                  onPressed: () => launchPaymentGateway("visa"),
                  child: Text("Pay via Visa/Mastercard"),
                )
              ],
            ),
          )
        ],
      ),
    );
  }
}

class TrackingScreen extends StatelessWidget {
  final double userLat;
  final double userLng;
  final double deliveryLat = -6.815; // Example delivery person location
  final double deliveryLng = 39.279;

  TrackingScreen({required this.userLat, required this.userLng});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Live Tracking")),
      body: GoogleMap(
        initialCameraPosition: CameraPosition(
          target: LatLng(userLat, userLng),
          zoom: 14,
        ),
        markers: {
          Marker(markerId: MarkerId("user"), position: LatLng(userLat, userLng), infoWindow: InfoWindow(title: "Your Location")),
          Marker(markerId: MarkerId("delivery"), position: LatLng(deliveryLat, deliveryLng), infoWindow: InfoWindow(title: "Delivery Guy")),
        },
      ),
    );
  }
}


