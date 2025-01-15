import SwiftUI
import MapKit

// MARK: - Custom Map Annotation
struct CustomMapAnnotation: Identifiable, Codable {
    let id: UUID
    var coordinate: CLLocationCoordinate2D
    var status: String // "available", "occupied", "out_of_service"
    var highlighted: Bool

    enum CodingKeys: String, CodingKey {
        case id, latitude, longitude, status, highlighted
    }

    init(id: UUID = UUID(), coordinate: CLLocationCoordinate2D, status: String, highlighted: Bool = false) {
        self.id = id
        self.coordinate = coordinate
        self.status = status
        self.highlighted = highlighted
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(UUID.self, forKey: .id)
        let latitude = try container.decode(CLLocationDegrees.self, forKey: .latitude)
        let longitude = try container.decode(CLLocationDegrees.self, forKey: .longitude)
        coordinate = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
        status = try container.decode(String.self, forKey: .status)
        highlighted = try container.decode(Bool.self, forKey: .highlighted)
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(coordinate.latitude, forKey: .latitude)
        try container.encode(coordinate.longitude, forKey: .longitude)
        try container.encode(status, forKey: .status)
        try container.encode(highlighted, forKey: .highlighted)
    }
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
            DispatchQueue.main.async {
                if let error = error {
                    self?.errorMessage = "Error: \(error.localizedDescription)"
                    return
                }
                guard let location = placemarks?.first?.location else {
                    self?.errorMessage = "No location found for query."
                    return
                }
                self?.setRegion(to: location.coordinate)
            }
        }
    }

    func setRegion(to coordinate: CLLocationCoordinate2D) {
        withAnimation {
            region = MKCoordinateRegion(
                center: coordinate,
                span: MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005)
            )
        }
    }
}

// MARK: - Parking Manager ViewModel
class ParkingManager: ObservableObject {
    @Published var isLoggedIn = false
    @Published var parkingMeters: [CustomMapAnnotation]
    @Published var showLoginSheet = false
    @Published var loginError: String?

    private let adminPassword = "telepark123"

    init() {
        if let data = UserDefaults.standard.data(forKey: "parkingMeters"),
           let savedMeters = try? JSONDecoder().decode([CustomMapAnnotation].self, from: data) {
            self.parkingMeters = savedMeters
        } else {
            self.parkingMeters = [
                CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6335, longitude: -71.3161), status: "available"),
                CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6340, longitude: -71.3155), status: "occupied"),
                CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6329, longitude: -71.3170), status: "out_of_service")
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
            loginError = nil
        } else {
            loginError = "Incorrect password. Please try again."
        }
    }

    func logout() {
        isLoggedIn = false
    }

    func highlightMeter(id: UUID) {
        if let index = parkingMeters.firstIndex(where: { $0.id == id }) {
            parkingMeters[index].highlighted.toggle()
            saveMeters()
        }
    }

    func addMeter(at coordinate: CLLocationCoordinate2D) {
        let newMeter = CustomMapAnnotation(coordinate: coordinate, status: "available")
        parkingMeters.append(newMeter)
        saveMeters()
    }
}

// MARK: - Main Content View
struct ContentView: View {
    @StateObject private var locationSearch = LocationSearch()
    @StateObject private var parkingManager = ParkingManager()
    @State private var searchQuery = ""
    @State private var password = ""

    var body: some View {
        NavigationView {
            VStack {
                // Header
                HStack {
                    Text("TelePark")
                        .font(.largeTitle)
                        .padding()
                    Spacer()
                    if parkingManager.isLoggedIn {
                        Button("Logout") {
                            parkingManager.logout()
                        }
                    }
                }

                // Search Bar
                TextField("Search for a location", text: $searchQuery)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding()
                    .onSubmit {
                        locationSearch.searchLocation(query: searchQuery)
                    }

                // Map View
                Map(coordinateRegion: $locationSearch.region, annotationItems: parkingManager.parkingMeters) { meter in
                    MapAnnotation(coordinate: meter.coordinate) {
                        VStack {
                            Circle()
                                .fill(meter.status == "available" ? (meter.highlighted ? Color.blue : Color.green) : Color.red)
                                .frame(width: 15, height: 15)
                                .onTapGesture {
                                    parkingManager.highlightMeter(id: meter.id)
                                }
                            Text(meter.status.capitalized)
                                .font(.caption)
                                .padding(4)
                                .background(Color.white.opacity(0.8))
                                .cornerRadius(5)
                        }
                    }
                }
                .frame(height: 300)

                Spacer()

                // Admin Actions
                if parkingManager.isLoggedIn {
                    Button("Add Meter Here") {
                        parkingManager.addMeter(at: locationSearch.region.center)
                    }
                    .padding()
                } else {
                    Button("Login as Admin") {
                        parkingManager.showLoginSheet = true
                    }
                    .padding()
                }
            }
            .sheet(isPresented: $parkingManager.showLoginSheet) {
                VStack {
                    Text("Admin Login")
                        .font(.headline)
                        .padding()

                    SecureField("Password", text: $password)
                        .textFieldStyle(RoundedBorderTextFieldStyle())
                        .padding()

                    if let error = parkingManager.loginError {
                        Text(error)
                            .foregroundColor(.red)
                            .padding(.bottom)
                    }

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

// MARK: - Preview
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
