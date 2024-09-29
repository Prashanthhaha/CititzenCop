import 'package:flutter/material.dart';
import 'package:camera/camera.dart';
import 'package:geocoding/geocoding.dart';
import 'dart:io';
import 'package:path_provider/path_provider.dart';
import 'package:geolocator/geolocator.dart';
import 'package:firebase_core/firebase_core.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  final cameras = await availableCameras();
  runApp(CameraApp(cameras: cameras));
}

class CameraApp extends StatelessWidget {
  final List<CameraDescription> cameras;

  const CameraApp({required this.cameras});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData.dark(),
      home: CameraScreen(cameras: cameras),
    );
  }
}

class CameraScreen extends StatefulWidget {
  final List<CameraDescription> cameras;

  const CameraScreen({required this.cameras});

  @override
  _CameraScreenState createState() => _CameraScreenState();
}

class _CameraScreenState extends State<CameraScreen> {
  CameraController? _controller;
  CameraDescription? _currentCamera;

  @override
  void initState() {
    super.initState();
    _currentCamera = widget.cameras.first;
    _initCamera(_currentCamera!);
  }

  Future<void> _initCamera(CameraDescription camera) async {
    _controller = CameraController(camera, ResolutionPreset.high);
    await _controller?.initialize();
    setState(() {});
  }

  @override
  void dispose() {
    _controller?.dispose();
    super.dispose();
  }

  Future<void> _takePicture() async {
    if (_controller != null && _controller!.value.isInitialized) {
      try {
        XFile picture = await _controller!.takePicture();

        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => PostCaptureScreen(imageFile: File(picture.path)),
          ),
        );
      } catch (e) {
        print('Error taking picture: $e');
      }
    }
  }

  void _switchCamera() {
    _currentCamera = _currentCamera == widget.cameras.first
        ? widget.cameras.last
        : widget.cameras.first;
    _initCamera(_currentCamera!);
  }

  @override
  Widget build(BuildContext context) {
    if (_controller == null || !_controller!.value.isInitialized) {
      return const Center(child: CircularProgressIndicator());
    }

    return Scaffold(
      body: GestureDetector(
        onHorizontalDragEnd: (details) {
          if (details.primaryVelocity != null && details.primaryVelocity! > 0) {
            Navigator.push(
              context,
              MaterialPageRoute(builder: (context) => StatusPage()),
            );
          }
        },
        child: Stack(
          children: [
            Positioned.fill(
              child: CameraPreview(_controller!),
            ),
            Positioned(
              bottom: 50,
              left: 0,
              right: 0,
              child: Center(
                child: GestureDetector(
                  onTap: _takePicture,
                  child: Container(
                    width: 80,
                    height: 80,
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      border: Border.all(color: Colors.white, width: 4),
                    ),
                  ),
                ),
              ),
            ),
            Positioned(
              top: 40,
              right: 20,
              child: IconButton(
                icon: const Icon(Icons.cameraswitch, color: Colors.white, size: 30),
                onPressed: _switchCamera,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class PostCaptureScreen extends StatefulWidget {
  final File imageFile;

  PostCaptureScreen({required this.imageFile});

  @override
  _PostCaptureScreenState createState() => _PostCaptureScreenState();
}

class _PostCaptureScreenState extends State<PostCaptureScreen> {
  String _currentLocation = 'Fetching location...';
  String _currentArea = 'Fetching area...';

  @override
  void initState() {
    super.initState();
    _getLocationAndArea();
  }

  Future<void> _getLocationAndArea() async {
    bool serviceEnabled;
    LocationPermission permission;

    serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) {
      setState(() {
        _currentLocation = 'Location services are disabled.';
        _currentArea = 'Unknown area';
      });
      return;
    }

    permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) {
        setState(() {
          _currentLocation = 'Location permission denied';
          _currentArea = 'Unknown area';
        });
        return;
      }
    }

    if (permission == LocationPermission.deniedForever) {
      setState(() {
        _currentLocation = 'Location permissions are permanently denied.';
        _currentArea = 'Unknown area';
      });
      return;
    }

    Position position = await Geolocator.getCurrentPosition(
        desiredAccuracy: LocationAccuracy.high);

    setState(() {
      _currentLocation = 'Lat: ${position.latitude}, Long: ${position.longitude}';
    });

    try {
      List<Placemark> placemarks = await placemarkFromCoordinates(
          position.latitude, position.longitude);

      Placemark place = placemarks[0];

      setState(() {
        _currentArea = '${place.locality}, ${place.administrativeArea}';
      });
    } catch (e) {
      setState(() {
        _currentArea = 'Unknown area';
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          Positioned.fill(
            child: Image.file(
              widget.imageFile,
              fit: BoxFit.cover,
            ),
          ),
          Positioned(
            bottom: 60,
            left: 20,
            right: 20,
            child: Text(
              _currentLocation,
              style: TextStyle(
                color: Colors.white,
                backgroundColor: Colors.black54,
                fontSize: 16,
              ),
              textAlign: TextAlign.center,
            ),
          ),
          Positioned(
            bottom: 40,
            left: 20,
            right: 20,
            child: Text(
              _currentArea,
              style: TextStyle(
                color: Colors.white,
                backgroundColor: Colors.black54,
                fontSize: 16,
              ),
              textAlign: TextAlign.center,
            ),
          ),
          Positioned(
            bottom: 90,
            left: 20,
            child: ElevatedButton.icon(
              onPressed: () {},
              icon: const Icon(Icons.save_alt),
              label: const Text('Save'),
              style: ElevatedButton.styleFrom(
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(10),
                ),
              ),
            ),
          ),
          Positioned(
            bottom: 90,
            right: 20,
            child: ElevatedButton(
              onPressed: () {
                Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => UploadScreen(
                      imagePath: widget.imageFile.path,
                      location: _currentLocation,
                      area: _currentArea,
                    ),
                  ),
                );
              },
              style: ElevatedButton.styleFrom(
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(10),
                ),
              ),
              child: const Text('Upload'),
            ),
          ),
        ],
      ),
    );
  }
}

class StatusPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Status & Payoff')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text(
              'Status',
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
                color: Colors.grey,
              ),
            ),
            const SizedBox(height: 10),
            const Text('1. Status 1: pending/fined/not fined'),
            const Text('2. Status 2'),
            Text('3. Status 3'),
            SizedBox(height: 20),
            Text(
              'Payoff',
              style: TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
                color: Colors.grey,
              ),
            ),
            SizedBox(height: 10),
            Text('Rs XX,XX.XX'),
            SizedBox(height: 10),
            ElevatedButton(
              onPressed: () {},
              child: Text('Redeem'),
            ),
          ],
        ),
      ),
    );
  }
}

class UploadScreen extends StatefulWidget {
  final String imagePath;
  final String location;
  final String area;

  UploadScreen({required this.imagePath, required this.location, required this.area});

  @override
  _UploadScreenState createState() => _UploadScreenState();
}

class _UploadScreenState extends State<UploadScreen> {
  String _selectedViolation = "Select violation";
  String _selectedArea = "Select area";
  String _selectedVehicleType = "Select vehicle type";

  List<String> violations = ['Select violation', 'Speeding', 'Parking Violation', 'Signal Jump'];
  List<String> areas = ['Select area', 'Downtown', 'Suburbs', 'Industrial'];
  List<String> vehicleTypes = ['Select vehicle type', 'Car', 'Bike', 'Truck'];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Upload Preview', style: TextStyle(fontWeight: FontWeight.bold)),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              crossAxisAlignment: CrossAxisAlignment.center,
              children: [
                Text(
                  'Image:',
                  style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                ),
                SizedBox(width: 10),
                GestureDetector(
                  onTap: () {
                    Navigator.push(
                      context,
                      MaterialPageRoute(
                        builder: (context) => FullImageScreen(
                          imagePath: widget.imagePath,
                          location: widget.location,
                          area: widget.area,
                        ),
                      ),
                    );
                  },
                  child: Container(
                    width: 50,
                    height: 50,
                    decoration: BoxDecoration(
                      border: Border.all(color: Colors.white),
                      color: Colors.grey[200],
                    ),
                    child: widget.imagePath.isNotEmpty
                        ? Image.file(
                            File(widget.imagePath),
                            fit: BoxFit.cover,
                          )
                        : Text('No image available'),
                  ),
                ),
              ],
            ),
            SizedBox(height: 20),
            Text(
              'Violation:',
              style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            DropdownButton<String>(
              value: _selectedViolation,
              isExpanded: true,
              items: violations.map((String value) {
                return DropdownMenuItem<String>(
                  value: value,
                  child: Text(value),
                );
              }).toList(),
              onChanged: (String? newValue) {
                setState(() {
                  _selectedViolation = newValue!;
                });
              },
            ),
            SizedBox(height: 20),
            Text(
              'Area:',
              style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            DropdownButton<String>(
              value: _selectedArea,
              isExpanded: true,
              items: areas.map((String value) {
                return DropdownMenuItem<String>(
                  value: value,
                  child: Text(value),
                );
              }).toList(),
              onChanged: (String? newValue) {
                setState(() {
                  _selectedArea = newValue!;
                });
              },
            ),
            SizedBox(height: 20),
            Text(
              'Vehicle Type:',
              style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            DropdownButton<String>(
              value: _selectedVehicleType,
              isExpanded: true,
              items: vehicleTypes.map((String value) {
                return DropdownMenuItem<String>(
                  value: value,
                  child: Text(value),
                );
              }).toList(),
              onChanged: (String? newValue) {
                setState(() {
                  _selectedVehicleType = newValue!;
                });
              },
            ),
            SizedBox(height: 40),
            Center(
              child: ElevatedButton(
                onPressed: () {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('Uploading...')),
                  );
                },
                child: Text('Upload'),
                style: ElevatedButton.styleFrom(
                  padding: EdgeInsets.symmetric(horizontal: 40, vertical: 15),
                  textStyle: TextStyle(fontSize: 16),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class FullImageScreen extends StatelessWidget {
  final String imagePath;
  final String location;
  final String area;

  FullImageScreen({required this.imagePath, required this.location, required this.area});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Full Image Preview'),
      ),
      body: Stack(
        children: [
          Positioned.fill(
            child: Image.file(
              File(imagePath),
              fit: BoxFit.cover,
            ),
          ),
          Positioned(
            bottom: 60,
            left: 20,
            right: 20,
            child: Text(
              location,
              style: TextStyle(
                color: Colors.white,
                backgroundColor: Colors.black54,
                fontSize: 16,
              ),
              textAlign: TextAlign.center,
            ),
          ),
          Positioned(
            bottom: 40,
            left: 20,
            right: 20,
            child: Text(
              area,
              style: TextStyle(
                color: Colors.white,
                backgroundColor: Colors.black54,
                fontSize: 16,
              ),
              textAlign: TextAlign.center,
            ),
          ),
        ],
      ),
    );
  }
}
