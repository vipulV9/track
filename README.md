package com.example.bustrackapp

import android.Manifest
import android.annotation.SuppressLint
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.location.LocationManager
import android.os.Build
import android.os.Bundle
import android.os.IBinder
import android.provider.Settings
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Clear
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.core.app.NotificationCompat
import androidx.core.content.ContextCompat
import com.google.android.gms.location.*
import com.google.firebase.database.DataSnapshot
import com.google.firebase.database.DatabaseError
import com.google.firebase.database.ValueEventListener
import com.google.firebase.database.ktx.database
import com.google.firebase.ktx.Firebase
import androidx.activity.compose.rememberLauncherForActivityResult
import android.content.SharedPreferences
import android.util.Log
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

data class BusRoute(
    val routeId: Int = 0,
    val latitude: Double = 0.0,
    val longitude: Double = 0.0
)

class MainActivity : ComponentActivity() {
    private lateinit var locationManager: LocationManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        locationManager = getSystemService(Context.LOCATION_SERVICE) as LocationManager
        Firebase.database.setPersistenceEnabled(true)
        setContent {
            BusRouteAppTheme {
                BusRouteApp(locationManager)
            }
        }
    }
}

@Composable
fun BusRouteAppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = lightColorScheme(
            primary = Color(0xFF1976D2),
            secondary = Color(0xFF4CAF50),
            background = Color(0xFFF5F5F5),
            error = Color(0xFFB00020)
        )
    ) {
        Surface(modifier = Modifier.fillMaxSize()) {
            content()
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BusRouteApp(locationManager: LocationManager) {
    val context = LocalContext.current
    val prefs = context.getSharedPreferences("BusTrackerPrefs", Context.MODE_PRIVATE)
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }

    var selectedRoute by remember { mutableStateOf(prefs.getInt("selectedRoute", 0)) }
    var latitude by remember { mutableStateOf(0.0) }
    var longitude by remember { mutableStateOf(0.0) }
    var isExpanded by remember { mutableStateOf(false) }
    var showGpsDialog by remember { mutableStateOf(false) }
    var showRouteConfirmDialog by remember { mutableStateOf(false) }
    var pendingRoute by remember { mutableStateOf(0) }
    var isTracking by remember { mutableStateOf(false) }
    var isLoading by remember { mutableStateOf(false) }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        val allGranted = permissions.values.all { it }
        if (allGranted) {
            Log.d("BusTracker", "All permissions granted")
            if (isGpsEnabled(locationManager)) {
                if (selectedRoute != 0 && !isTracking) {
                    pendingRoute = selectedRoute
                    showRouteConfirmDialog = true
                }
            } else {
                showGpsDialog = true
            }
        } else {
            scope.launch {
                snackbarHostState.showSnackbar("Location permission denied. Cannot track bus.")
            }
        }
    }

    LaunchedEffect(Unit) {
        val permissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION,
                Manifest.permission.ACCESS_BACKGROUND_LOCATION
            )
        } else {
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION
            )
        }
        if (permissions.any { ContextCompat.checkSelfPermission(context, it) != PackageManager.PERMISSION_GRANTED }) {
            permissionLauncher.launch(permissions)
        } else if (!isGpsEnabled(locationManager)) {
            showGpsDialog = true
        } else if (selectedRoute != 0 && !isTracking) {
            pendingRoute = selectedRoute
            showRouteConfirmDialog = true
        }
    }

    if (showGpsDialog) {
        AlertDialog(
            onDismissRequest = { showGpsDialog = false },
            title = { Text("Enable GPS") },
            text = { Text("GPS is required for accurate tracking. Please enable it.") },
            confirmButton = {
                TextButton(onClick = {
                    context.startActivity(Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS))
                    showGpsDialog = false
                }) {
                    Text("Settings")
                }
            },
            dismissButton = {
                TextButton(onClick = { showGpsDialog = false }) {
                    Text("Cancel")
                }
            }
        )
    }

    if (showRouteConfirmDialog) {
        AlertDialog(
            onDismissRequest = { showRouteConfirmDialog = false },
            title = { Text("Confirm Route") },
            text = { Text("Send location for Route $pendingRoute?") },
            confirmButton = {
                TextButton(onClick = {
                    isLoading = true
                    selectedRoute = pendingRoute
                    prefs.edit().putInt("selectedRoute", selectedRoute).apply()
                    updateServiceRoute(context, selectedRoute)
                    isTracking = true
                    showRouteConfirmDialog = false
                    scope.launch {
                        delay(1000)
                        isLoading = false
                    }
                    Log.d("BusTracker", "Route $selectedRoute confirmed")
                }) {
                    Text("Yes")
                }
            },
            dismissButton = {
                TextButton(onClick = {
                    showRouteConfirmDialog = false
                    Log.d("BusTracker", "Route $pendingRoute cancelled")
                }) {
                    Text("No")
                }
            }
        )
    }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .background(MaterialTheme.colorScheme.background)
                .padding(paddingValues)
                .padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(
                text = "Bus Tracker",
                fontSize = 32.sp,
                fontWeight = FontWeight.Bold,
                color = MaterialTheme.colorScheme.primary,
                modifier = Modifier.padding(bottom = 16.dp)
            )

            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .clip(RoundedCornerShape(16.dp))
                    .border(1.dp, Color.LightGray, RoundedCornerShape(16.dp)),
                elevation = CardDefaults.cardElevation(8.dp)
            ) {
                Column(
                    modifier = Modifier
                        .padding(16.dp)
                        .fillMaxWidth()
                ) {
                    Text(
                        text = "Select Your Bus Route",
                        fontSize = 18.sp,
                        fontWeight = FontWeight.Medium,
                        color = MaterialTheme.colorScheme.primary
                    )

                    Spacer(modifier = Modifier.height(12.dp))

                    ExposedDropdownMenuBox(
                        expanded = isExpanded,
                        onExpandedChange = { isExpanded = it },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        OutlinedTextField(
                            value = if (selectedRoute == 0) "Choose a Route" else "Route $selectedRoute",
                            onValueChange = {},
                            readOnly = true,
                            label = { Text("Bus Route") },
                            trailingIcon = {
                                ExposedDropdownMenuDefaults.TrailingIcon(expanded = isExpanded)
                            },
                            modifier = Modifier
                                .fillMaxWidth()
                                .menuAnchor(),
                            colors = TextFieldDefaults.outlinedTextFieldColors(
                                focusedBorderColor = MaterialTheme.colorScheme.primary,
                                cursorColor = MaterialTheme.colorScheme.primary
                            )
                        )

                        ExposedDropdownMenu(
                            expanded = isExpanded,
                            onDismissRequest = { isExpanded = false },
                            modifier = Modifier
                                .background(Color.White)
                                .heightIn(max = 300.dp)
                        ) {
                            (1..60).forEach { route ->
                                DropdownMenuItem(
                                    text = { Text("Route $route") },
                                    onClick = {
                                        isExpanded = false
                                        pendingRoute = route
                                        val permissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                                            arrayOf(
                                                Manifest.permission.ACCESS_FINE_LOCATION,
                                                Manifest.permission.ACCESS_BACKGROUND_LOCATION
                                            )
                                        } else {
                                            arrayOf(Manifest.permission.ACCESS_FINE_LOCATION)
                                        }
                                        if (permissions.all { ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED } &&
                                            isGpsEnabled(locationManager)) {
                                            showRouteConfirmDialog = true
                                        } else if (!permissions.all { ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED }) {
                                            permissionLauncher.launch(permissions)
                                        } else {
                                            showGpsDialog = true
                                        }
                                    },
                                    modifier = Modifier
                                        .background(if (route == selectedRoute) MaterialTheme.colorScheme.secondary.copy(alpha = 0.1f) else Color.White)
                                        .fillMaxWidth()
                                )
                            }
                        }
                    }
                }
            }

            Spacer(modifier = Modifier.height(24.dp))

            AnimatedVisibility(visible = isTracking) {
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .clip(RoundedCornerShape(16.dp)),
                    elevation = CardDefaults.cardElevation(8.dp),
                    colors = CardDefaults.cardColors(containerColor = Color.White)
                ) {
                    Row(
                        modifier = Modifier
                            .padding(16.dp)
                            .fillMaxWidth(),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        Icon(
                            imageVector = Icons.Default.LocationOn,
                            contentDescription = "Location",
                            tint = MaterialTheme.colorScheme.secondary,
                            modifier = Modifier.size(32.dp)
                        )

                        Spacer(modifier = Modifier.width(16.dp))

                        Column(modifier = Modifier.weight(1f)) {
                            Text(
                                text = "Current Location",
                                fontSize = 18.sp,
                                fontWeight = FontWeight.Medium,
                                color = MaterialTheme.colorScheme.primary
                            )
                            Spacer(modifier = Modifier.height(8.dp))
                            Text(
                                text = "Lat: ${String.format("%.6f", latitude)}",
                                fontSize = 16.sp,
                                color = Color.Gray
                            )
                            Text(
                                text = "Long: ${String.format("%.6f", longitude)}",
                                fontSize = 16.sp,
                                color = Color.Gray
                            )
                        }

                        IconButton(onClick = {
                            stopServiceRoute(context)
                            selectedRoute = 0
                            prefs.edit().putInt("selectedRoute", 0).apply()
                            isTracking = false
                            latitude = 0.0
                            longitude = 0.0
                        }) {
                            Icon(
                                imageVector = Icons.Default.Clear,
                                contentDescription = "Stop Tracking",
                                tint = MaterialTheme.colorScheme.error
                            )
                        }
                    }
                }
            }

            Spacer(modifier = Modifier.height(16.dp))

            AnimatedVisibility(visible = isLoading) {
                CircularProgressIndicator(
                    color = MaterialTheme.colorScheme.primary,
                    modifier = Modifier.size(32.dp)
                )
            }

            Spacer(modifier = Modifier.weight(1f))

            Text(
                text = when {
                    isLoading -> "Starting tracking..."
                    isTracking -> "Tracking Route $selectedRoute"
                    else -> "Please select a route to start tracking"
                },
                fontSize = 14.sp,
                color = if (!isTracking) Color.Gray else MaterialTheme.colorScheme.secondary,
                textAlign = TextAlign.Center,
                modifier = Modifier.padding(bottom = 16.dp)
            )
        }
    }

    DisposableEffect(selectedRoute) {
        if (selectedRoute == 0) return@DisposableEffect onDispose { }
        val database = Firebase.database
        val routeRef = database.getReference("bus_routes/$selectedRoute")
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) {
                try {
                    val busRoute = snapshot.getValue(BusRoute::class.java)
                    busRoute?.let {
                        latitude = it.latitude
                        longitude = it.longitude
                        Log.d("BusTracker", "UI updated: ${it.latitude}, ${it.longitude}")
                    } ?: run {
                        Log.w("BusTracker", "No data for route $selectedRoute")
                    }
                } catch (e: Exception) {
                    Log.e("BusTracker", "Error parsing Firebase data: ${e.message}")
                    scope.launch {
                        snackbarHostState.showSnackbar("Error fetching location data")
                    }
                }
            }

            override fun onCancelled(error: DatabaseError) {
                Log.e("BusTracker", "Firebase error: ${error.message}")
                scope.launch {
                    snackbarHostState.showSnackbar("Database error: ${error.message}")
                }
            }
        }
        routeRef.addValueEventListener(listener)
        onDispose {
            routeRef.removeEventListener(listener)
        }
    }
}

private fun isGpsEnabled(locationManager: LocationManager): Boolean {
    return try {
        locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER)
    } catch (e: Exception) {
        Log.e("BusTracker", "Error checking GPS: ${e.message}")
        false
    }
}

private fun updateServiceRoute(context: Context, routeId: Int) {
    val intent = Intent(context, LocationService::class.java).apply {
        putExtra("routeId", routeId)
        action = "UPDATE_ROUTE"
    }
    try {
        context.startForegroundService(intent)
        Log.d("BusTracker", "Service started for route $routeId")
    } catch (e: Exception) {
        Log.e("BusTracker", "Error starting service: ${e.message}")
    }
}

private fun stopServiceRoute(context: Context) {
    try {
        context.stopService(Intent(context, LocationService::class.java))
        Log.d("BusTracker", "Service stopped")
    } catch (e: Exception) {
        Log.e("BusTracker", "Error stopping service: ${e.message}")
    }
}

class LocationService : Service() {
    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private lateinit var locationCallback: LocationCallback
    private var currentRouteId: Int = 0

    companion object {
        private const val NOTIFICATION_ID = 1
        private const val CHANNEL_ID = "location_channel"
    }

    override fun onCreate() {
        super.onCreate()
        try {
            fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
            createNotificationChannel()
            Log.d("BusTracker", "Service created")
        } catch (e: Exception) {
            Log.e("BusTracker", "Service creation failed: ${e.message}")
            stopSelf()
        }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        intent?.let {
            if (it.action == "UPDATE_ROUTE") {
                currentRouteId = it.getIntExtra("routeId", 0)
                Log.d("BusTracker", "Service started with route $currentRouteId")
            }
        } ?: Log.w("BusTracker", "Null intent received")

        if (currentRouteId == 0) {
            stopSelf()
            return START_NOT_STICKY
        }

        val notification = createNotification(currentRouteId)
        startForeground(NOTIFICATION_ID, notification.build())
        startLocationUpdates()
        return START_NOT_STICKY // Changed to NOT_STICKY
    }

    @SuppressLint("MissingPermission")
    private fun startLocationUpdates() {
        val locationRequest = LocationRequest.create().apply {
            interval = 5000
            fastestInterval = 2000
            priority = LocationRequest.PRIORITY_HIGH_ACCURACY
        }

        locationCallback = object : LocationCallback() {
            override fun onLocationResult(locationResult: LocationResult) {
                locationResult.lastLocation?.let { location ->
                    Log.d("BusTracker", "Location: ${location.latitude}, ${location.longitude}")
                    updateLocationInFirebase(currentRouteId, location.latitude, location.longitude)
                } ?: Log.w("BusTracker", "No location available")
            }
        }

        try {
            if (ContextCompat.checkSelfPermission(
                    this,
                    Manifest.permission.ACCESS_FINE_LOCATION
                ) == PackageManager.PERMISSION_GRANTED) {
                fusedLocationClient.removeLocationUpdates(locationCallback)
                fusedLocationClient.requestLocationUpdates(locationRequest, locationCallback, null)
                Log.d("BusTracker", "Location updates started")
            } else {
                Log.e("BusTracker", "No location permission")
                stopSelf()
            }
        } catch (e: Exception) {
            Log.e("BusTracker", "Error starting location updates: ${e.message}")
            stopSelf()
        }
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "Location Updates",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "Shows bus tracking status"
            }
            val manager = getSystemService(NotificationManager::class.java)
            manager?.createNotificationChannel(channel)
        }
    }

    private fun createNotification(routeId: Int): NotificationCompat.Builder {
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            this,
            0,
            intent,
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) PendingIntent.FLAG_IMMUTABLE else 0
        )

        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Bus Tracking Active")
            .setContentText("Tracking Route $routeId")
            .setSmallIcon(android.R.drawable.ic_dialog_map)
            .setContentIntent(pendingIntent)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setOngoing(true)
    }

    private fun updateLocationInFirebase(routeId: Int, latitude: Double, longitude: Double) {
        try {
            val database = Firebase.database
            val routeRef = database.getReference("bus_routes/$routeId")
            val busRoute = BusRoute(routeId, latitude, longitude)
            routeRef.setValue(busRoute)
                .addOnSuccessListener {
                    Log.d("BusTracker", "Firebase updated: $routeId, $latitude, $longitude")
                }
                .addOnFailureListener { e ->
                    Log.e("BusTracker", "Firebase update failed: ${e.message}")
                }
        } catch (e: Exception) {
            Log.e("BusTracker", "Error updating Firebase: ${e.message}")
        }
    }

    override fun onTaskRemoved(rootIntent: Intent?) {
        super.onTaskRemoved(rootIntent)
        Log.d("BusTracker", "App removed from recent apps, stopping service")
        stopSelf() // Stop the service when app is swiped away
    }

    override fun onDestroy() {
        super.onDestroy()
        try {
            fusedLocationClient.removeLocationUpdates(locationCallback)
            Log.d("BusTracker", "Service destroyed")
        } catch (e: Exception) {
            Log.e("BusTracker", "Error destroying service: ${e.message}")
        }
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
