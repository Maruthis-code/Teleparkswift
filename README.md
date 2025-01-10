import SwiftUI
import MapKit

// MARK: - Custom Annotation Struct
struct CustomMapAnnotation: Identifiable, Codable {
    let id = UUID()
    let coordinate: CLLocationCoordinate2D
    var status: String // Available, Occupied, Out of Service
}

// MARK: - Location Search ViewModel
class LocationSearch: ObservableObject {
    @Published var region: MKCoordinateRegion
    @Published var errorMessage: String?
    
    init() {
        self.region = MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 42.6334, longitude: -71.3162),
            span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)
        )
    }

    func searchLocation(query: String) {
        let geocoder = CLGeocoder()
        geocoder.geocodeAddressString(query) { [weak self] placemarks, error in
            if let error = error {
                DispatchQueue.main.async {
                    self?.errorMessage = "Error: \(error.localizedDescription)"
                }
                return
            }
            guard let location = placemarks?.first?.location else {
                DispatchQueue.main.async {
                    self?.errorMessage = "No location found for query."
                }
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

// MARK: - Parking Manager ViewModel
class ParkingManager: ObservableObject {
    @Published var isLoggedIn = false
    @Published var showLoginSheet = false
    @Published var parkingMeters: [CustomMapAnnotation]
    @Published var showParkingComplete = false
    private let adminPassword = "telepark123"

    init() {
        if let data = UserDefaults.standard.data(forKey: "parkingMeters"),
           let savedMeters = try? JSONDecoder().decode([CustomMapAnnotation].self, from: data) {
            self.parkingMeters = savedMeters
        } else {
            self.parkingMeters = [
                CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6335, longitude: -71.3161), status: "available"),
                CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6340, longitude: -71.3155), status: "occupied"),
                CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6329, longitude: -71.3170), status: "available")
            ]
        }
    }

    func saveMeters() {
        if let encoded = try? JSONEncoder().encode(parkingMeters) {
            UserDefaults.standard.set(encoded, forKey: "parkingMeters")
        }
    }

    func login(password: String) {
        if password == adminPassword {
            isLoggedIn = true
            showLoginSheet = false
        }
    }

    func updateMeterStatus(id: UUID, status: String) {
        if let index = parkingMeters.firstIndex(where: { $0.id == id }) {
            parkingMeters[index].status = status
            saveMeters()
        }
    }

    func findNearestAvailableMeter(from location: CLLocationCoordinate2D) -> CustomMapAnnotation? {
        let userLocation = CLLocation(latitude: location.latitude, longitude: location.longitude)
        return parkingMeters
            .filter { $0.status == "available" }
            .min(by: {
                let meterLocation1 = CLLocation(latitude: $0.coordinate.latitude, longitude: $0.coordinate.longitude)
                let meterLocation2 = CLLocation(latitude: $1.coordinate.latitude, longitude: $1.coordinate.longitude)
                return userLocation.distance(from: meterLocation1) < userLocation.distance(from: meterLocation2)
            })
    }
}

// MARK: - Content View
struct ContentView: View {
    @StateObject private var locationSearch = LocationSearch()
    @StateObject private var parkingManager = ParkingManager()
    @State private var searchQuery = ""
    @State private var password = ""
    @State private var showFeedbackForm = false

    var body: some View {
        NavigationView {
            VStack {
                Text("TelePark")
                    .font(.largeTitle)
                    .padding()

                TextField("Search for a location", text: $searchQuery)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding()
                    .onSubmit {
                        locationSearch.searchLocation(query: searchQuery)
                    }

                Map(coordinateRegion: $locationSearch.region, annotationItems: parkingManager.parkingMeters) { meter in
                    Annotation("Parking Meter", coordinate: meter.coordinate) {
                        VStack {
                            Circle()
                                .fill(meter.status == "available" ? Color.green : meter.status == "occupied" ? Color.red : Color.gray)
                                .frame(width: 10, height: 10)
                                .onTapGesture {
                                    if parkingManager.isLoggedIn {
                                        let newStatus = meter.status == "available" ? "occupied" : "available"
                                        parkingManager.updateMeterStatus(id: meter.id, status: newStatus)
                                    }
                                }
                            Text(meter.status.capitalized)
                                .font(.caption)
                                .background(Color.white)
                                .cornerRadius(5)
                        }
                    }
                }
                .frame(height: 300)

                Button("Find Nearest Available Spot") {
                    if let nearest = parkingManager.findNearestAvailableMeter(from: locationSearch.region.center) {
                        locationSearch.region.center = nearest.coordinate
                    } else {
                        locationSearch.errorMessage = "No available parking spots nearby!"
                    }
                }
                .padding()

                if let errorMessage = locationSearch.errorMessage {
                    Text(errorMessage)
                        .foregroundColor(.red)
                        .padding()
                }

                Button("Provide Feedback") {
                    showFeedbackForm = true
                }
                .sheet(isPresented: $showFeedbackForm) {
                    FeedbackFormView()
                }

                Spacer()

                if !parkingManager.isLoggedIn {
                    Button("Parking Director Login") {
                        parkingManager.showLoginSheet = true
                    }
                } else {
                    Button("Logout") {
                        parkingManager.isLoggedIn = false
                    }
                }
            }
        }
        .sheet(isPresented: $parkingManager.showLoginSheet) {
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
            }
        }
    }
}

// MARK: - Feedback Form View
struct FeedbackFormView: View {
    var body: some View {
        VStack {
            Text("Feedback Form")
                .font(.largeTitle)
                .padding()
            Link("Open Feedback Form", destination: URL(string: "https://forms.gle/example")!)
        }
    }
}
