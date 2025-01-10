import SwiftUI
import MapKit

// MARK: - Custom Map Annotation
struct CustomMapAnnotation: Identifiable, Codable {
    let id: UUID
    var coordinate: CLLocationCoordinate2D
    var status: String // "available", "occupied", "out_of_service"

    // Coding keys for manual encoding/decoding
    enum CodingKeys: String, CodingKey {
        case id
        case latitude
        case longitude
        case status
    }

    // Decoding: Reconstruct CLLocationCoordinate2D
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(UUID.self, forKey: .id)
        let latitude = try container.decode(CLLocationDegrees.self, forKey: .latitude)
        let longitude = try container.decode(CLLocationDegrees.self, forKey: .longitude)
        coordinate = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
        status = try container.decode(String.self, forKey: .status)
    }

    // Encoding: Serialize CLLocationCoordinate2D into latitude and longitude
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(coordinate.latitude, forKey: .latitude)
        try container.encode(coordinate.longitude, forKey: .longitude)
        try container.encode(status, forKey: .status)
    }

    // Default initializer
    init(id: UUID = UUID(), coordinate: CLLocationCoordinate2D, status: String) {
        self.id = id
        self.coordinate = coordinate
        self.status = status
    }
}

// MARK: - Location Search ViewModel
class LocationSearch: ObservableObject {
    @Published var region: MKCoordinateRegion
    @Published var errorMessage: String?
    @Published var recentSearches: [String] = [] // Store search history

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
                self?.setRegion(to: location.coordinate)
                self?.addToRecentSearches(query: query)
            }
        }
    }

    func setRegion(to coordinate: CLLocationCoordinate2D) {
        withAnimation {
            self.region = MKCoordinateRegion(
                center: coordinate,
                span: MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005) // Zoom in tightly
            )
        }
    }

    private func addToRecentSearches(query: String) {
        if !recentSearches.contains(query) {
            recentSearches.insert(query, at: 0) // Add the latest search to the top
        }
        if recentSearches.count > 5 {
            recentSearches.removeLast() // Keep only the last 5 searches
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
                CustomMapAnnotation(id: UUID(), coordinate: CLLocationCoordinate2D(latitude: 42.6335, longitude: -71.3161), status: "available"),
                CustomMapAnnotation(id: UUID(), coordinate: CLLocationCoordinate2D(latitude: 42.6340, longitude: -71.3155), status: "occupied"),
                CustomMapAnnotation(id: UUID(), coordinate: CLLocationCoordinate2D(latitude: 42.6329, longitude: -71.3170), status: "available")
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

    func addMeter(at coordinate: CLLocationCoordinate2D) {
        let newMeter = CustomMapAnnotation(id: UUID(), coordinate: coordinate, status: "available")
        parkingMeters.append(newMeter)
        saveMeters()
    }

    func moveMeter(id: UUID, to coordinate: CLLocationCoordinate2D) {
        if let index = parkingMeters.firstIndex(where: { $0.id == id }) {
            parkingMeters[index].coordinate = coordinate
            saveMeters()
        }
    }
}

// MARK: - Main Content View
struct ContentView: View {
    @StateObject private var locationSearch = LocationSearch()
    @StateObject private var parkingManager = ParkingManager()
    @State private var searchQuery = ""
    @State private var password = ""
    @State private var showFeedbackForm = false
    @State private var selectedMeter: CustomMapAnnotation?

    var body: some View {
        NavigationView {
            VStack {
                // Header
                Text("TelePark")
                    .font(.largeTitle)
                    .padding()

                Divider()

                // Search Bar and Recent Searches
                VStack(alignment: .leading) {
                    TextField("Search for a location", text: $searchQuery)
                        .textFieldStyle(RoundedBorderTextFieldStyle())
                        .padding(.horizontal)
                        .onSubmit {
                            locationSearch.searchLocation(query: searchQuery)
                        }

                    if !locationSearch.recentSearches.isEmpty {
                        Text("Recent Searches:")
                            .font(.headline)
                            .padding(.horizontal)
                        ScrollView(.horizontal, showsIndicators: false) {
                            HStack {
                                ForEach(locationSearch.recentSearches, id: \.self) { search in
                                    Button(action: {
                                        locationSearch.searchLocation(query: search)
                                    }) {
                                        Text(search)
                                            .padding(8)
                                            .background(Color.gray.opacity(0.2))
                                            .cornerRadius(10)
                                    }
                                }
                            }
                            .padding(.horizontal)
                        }
                    }
                }

                Divider()

                // Map View
                Map(coordinateRegion: $locationSearch.region, interactionModes: [.all], annotationItems: parkingManager.parkingMeters) { meter in
                    MapAnnotation(coordinate: meter.coordinate) {
                        VStack {
                            Circle()
                                .fill(meter.status == "available" ? Color.green : meter.status == "occupied" ? Color.red : Color.gray)
                                .frame(width: 10, height: 10)
                                .onTapGesture {
                                    if parkingManager.isLoggedIn {
                                        selectedMeter = meter
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
                .frame(height: 300)

                // Actions
                Button("Find Nearest Available Spot") {
                    if let nearest = parkingManager.parkingMeters.filter({ $0.status == "available" }).first {
                        locationSearch.setRegion(to: nearest.coordinate)
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

                Spacer()

                if parkingManager.isLoggedIn {
                    Button("Add Meter Here") {
                        parkingManager.addMeter(at: locationSearch.region.center)
                    }
                    .padding()
                }

                Button(parkingManager.isLoggedIn ? "Logout" : "Login as Parking Director") {
                    parkingManager.showLoginSheet = true
                }
                .padding()

                Spacer()
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
