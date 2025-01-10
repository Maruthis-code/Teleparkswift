import SwiftUI
import MapKit

// Protocol Definition
protocol MapAnnotationProtocol {
    var coordinate: CLLocationCoordinate2D { get }
}

// Custom Annotation Struct
struct CustomMapAnnotation: Identifiable, MapAnnotationProtocol {
    let id = UUID()
    let coordinate: CLLocationCoordinate2D
    var status: String // Available, Occupied, Out of Service
}

// ObservableObject for Location Search
class LocationSearch: ObservableObject {
    @Published var region = MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 42.6334, longitude: -71.3162),
        span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)
    )
    @Published var errorMessage: String?

    func searchLocation(query: String) {
        let geocoder = CLGeocoder()
        geocoder.geocodeAddressString(query) { [weak self] placemarks, error in
            if let error = error {
                self?.errorMessage = "Error searching for location: \(error.localizedDescription)"
                return
            }
            guard let location = placemarks?.first?.location else {
                self?.errorMessage = "No location found"
                return
            }
            DispatchQueue.main.async {
                withAnimation {
                    self?.region.center = location.coordinate
                }
            }
        }
    }
}

// ViewModel for Managing Login and Parking Meters
class ParkingManager: ObservableObject {
    @Published var isLoggedIn = false
    @Published var showLogin = false
    @Published var parkingMeters: [CustomMapAnnotation] = [
        CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6335, longitude: -71.3161), status: "available"),
        CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6340, longitude: -71.3155), status: "occupied"),
        CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6329, longitude: -71.3170), status: "available")
    ]
    @Published var showParkingComplete = false
    let adminPassword = "telepark123"

    func login(password: String) {
        if password == adminPassword {
            isLoggedIn = true
            showLogin = false
        }
    }

    func updateMeterStatus(id: UUID, status: String) {
        if let index = parkingMeters.firstIndex(where: { $0.id == id }) {
            parkingMeters[index].status = status
        }
    }

    func findNearestAvailableMeter(from location: CLLocationCoordinate2D) -> CustomMapAnnotation? {
        return parkingMeters
            .filter { $0.status == "available" }
            .min(by: { CLLocation(latitude: $0.coordinate.latitude, longitude: $0.coordinate.longitude).distance(from: CLLocation(latitude: location.latitude, longitude: location.longitude)) < CLLocation(latitude: $1.coordinate.latitude, longitude: $1.coordinate.longitude)) })
    }
}

// Main Content View
struct ContentView: View {
    @StateObject var locationSearch = LocationSearch()
    @StateObject var parkingManager = ParkingManager()
    @State private var searchQuery = "UMass Lowell"
    @State private var password = ""

    var body: some View {
        NavigationView {
            VStack {
                // Search Bar
                TextField("Search for a location", text: $searchQuery)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding()
                    .onSubmit {
                        locationSearch.searchLocation(query: searchQuery)
                    }

                // Map
                Map(coordinateRegion: $locationSearch.region, annotationItems: parkingManager.parkingMeters) { meter in
                    Annotation("Parking Meter", coordinate: meter.coordinate) {
                        VStack {
                            Circle()
                                .fill(meter.status == "available" ? Color.green : meter.status == "occupied" ? Color.red : Color.gray)
                                .frame(width: 10, height: 10)
                                .onTapGesture {
                                    if parkingManager.isLoggedIn {
                                        parkingManager.updateMeterStatus(id: meter.id, status: meter.status == "available" ? "occupied" : "available")
                                    }
                                }

                            Text(meter.status.capitalized)
                                .font(.caption)
                                .padding(4)
                                .background(Color.white)
                                .cornerRadius(5)
                        }
                    }
                }
                .ignoresSafeArea()

                // Nearest Available Spot Button
                Button("Find Nearest Available Spot") {
                    if let nearest = parkingManager.findNearestAvailableMeter(from: locationSearch.region.center) {
                        locationSearch.region.center = nearest.coordinate
                    } else {
                        locationSearch.errorMessage = "No available parking spots nearby!"
                    }
                }
                .padding()

                // Display Error Message
                if let errorMessage = locationSearch.errorMessage {
                    Text(errorMessage)
                        .foregroundColor(.red)
                        .padding()
                }

                Spacer()

                // Login for Parking Directors
                if !parkingManager.isLoggedIn {
                    Button("Parking Director Login") {
                        parkingManager.showLogin = true
                    }
                    .padding()
                }

                // Logout for Parking Directors
                if parkingManager.isLoggedIn {
                    Button("Logout") {
                        parkingManager.isLoggedIn = false
                    }
                    .padding()
                }

                NavigationLink(destination: ParkingCompleteView(), isActive: $parkingManager.showParkingComplete) {
                    EmptyView()
                }
            }
            .sheet(isPresented: $parkingManager.showLogin) {
                VStack {
                    Text("Enter Admin Password")
                        .font(.headline)
                        .padding()
                    SecureField("Password", text: $password)
                        .textFieldStyle(RoundedBorderTextFieldStyle())
                        .padding()
                    Button("Login") {
                        parkingManager.login(password: password)
                        password = ""
                    }
                    .padding()
                }
                .padding()
            }
        }
    }
}

// Parking Complete View
struct ParkingCompleteView: View {
    var body: some View {
        VStack {
            Text("Parking Complete!")
                .font(.largeTitle)
                .padding()

            Link("Provide Feedback", destination: URL(string: "https://forms.gle/example")!)
                .padding()
        }
    }
}

// Preview
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
