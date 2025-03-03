import 'package:flutter/material.dart';
import 'package:qr_code_scanner/qr_code_scanner.dart';
import 'package:flutter_stripe/flutter_stripe.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Payment Scanner',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: QRScannerPage(),
    );
  }
}

class QRScannerPage extends StatefulWidget {
  @override
  _QRScannerPageState createState() => _QRScannerPageState();
}

class _QRScannerPageState extends State<QRScannerPage> {
  final GlobalKey<QRViewControllerState> _qrKey = GlobalKey();
  QRViewController? _controller;
  String? _scannedData;
  bool _isProcessingPayment = false;

  @override
  void dispose() {
    _controller?.dispose();
    super.dispose();
  }

  // Function to initiate payment
  Future<void> makePayment(String amount) async {
    setState(() {
      _isProcessingPayment = true;
    });

    try {
      // Assuming you have a backend API that creates a PaymentIntent and returns the client secret
      final clientSecret = await createPaymentIntent(amount); // Replace with your backend API call

      // Initialize Stripe
      Stripe.publishableKey = 'your-publishable-key';
      await Stripe.instance.initPaymentSheet(
        paymentSheetParameters: SetupPaymentSheetParameters(
          paymentIntentClientSecret: clientSecret,
          merchantDisplayName: 'Merchant Name',
        ),
      );

      // Show Payment Sheet
      await Stripe.instance.presentPaymentSheet();
      setState(() {
        _isProcessingPayment = false;
      });

      // Display success message
      showDialog(
        context: context,
        builder: (context) => AlertDialog(
          title: Text('Payment Successful'),
          content: Text('Your payment of \$${amount} was successful!'),
          actions: <Widget>[
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: Text('OK'),
            ),
          ],
        ),
      );
    } catch (e) {
      setState(() {
        _isProcessingPayment = false;
      });
      // Handle payment error
      showDialog(
        context: context,
        builder: (context) => AlertDialog(
          title: Text('Payment Failed'),
          content: Text('There was an issue processing your payment: $e'),
          actions: <Widget>[
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: Text('OK'),
            ),
          ],
        ),
      );
    }
  }

  // Simulating a backend call to get the client secret for Stripe payment
  Future<String> createPaymentIntent(String amount) async {
    // Replace with actual backend API call to create payment intent
    await Future.delayed(Duration(seconds: 2)); // Simulate network delay
    return 'your-client-secret'; // This should come from your backend
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Scan QR Code for Payment'),
      ),
      body: Column(
        children: [
          Expanded(
            child: QRView(
              key: _qrKey,
              onQRViewCreated: (controller) {
                setState(() {
                  _controller = controller;
                });
                controller.scannedDataStream.listen((scanData) {
                  setState(() {
                    _scannedData = scanData.code;
                  });
                  if (_scannedData != null) {
                    // After scanning, proceed with payment
                    makePayment('20'); // You can parse the amount from the QR code
                  }
                });
              },
            ),
          ),
          if (_isProcessingPayment)
            CircularProgressIndicator(),
          if (!_isProcessingPayment && _scannedData != null)
            Padding(
              padding: const EdgeInsets.all(16.0),
              child: Text(
                'Scanned Payment QR Code: $_scannedData',
                style: TextStyle(fontSize: 16),
              ),
            ),
        ],
      ),
    );
  }
}
