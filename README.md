import SwiftUI
import MapKit

// MARK: - Custom Map Annotation
struct CustomMapAnnotation: Identifiable, Codable {
    let id: UUID
    var coordinate: CLLocationCoordinate2D
    var status: String // "available", "occupied", "out_of_service"

    enum CodingKeys: String, CodingKey {
        case id
        case latitude
        case longitude
        case status
    }

    init(id: UUID = UUID(), latitude: CLLocationDegrees, longitude: CLLocationDegrees, status: String) {
        self.id = id
        self.coordinate = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
        self.status = status
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(UUID.self, forKey: .id)
        let latitude = try container.decode(CLLocationDegrees.self, forKey: .latitude)
        let longitude = try container.decode(CLLocationDegrees.self, forKey: .longitude)
        coordinate = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
        status = try container.decode(String.self, forKey: .status)
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(coordinate.latitude, forKey: .latitude)
        try container.encode(coordinate.longitude, forKey: .longitude)
        try container.encode(status, forKey: .status)
    }
}

// MARK: - Extensions
extension Double {
    var degreesToRadians: Double {
        return self * .pi / 180
    }
}

// MARK: - Haversine Formula
func haversineDistance(lat1: Double, lon1: Double, lat2: Double, lon2: Double) -> Double {
    let radiusEarthKm = 6371.0
    let dLat = (lat2 - lat1).degreesToRadians
    let dLon = (lon2 - lon1).degreesToRadians
    let a = sin(dLat / 2) * sin(dLat / 2) +
        cos(lat1.degreesToRadians) * cos(lat2.degreesToRadians) *
        sin(dLon / 2) * sin(dLon / 2)
    let c = 2 * atan2(sqrt(a), sqrt(1 - a))
    return radiusEarthKm * c
}

// MARK: - Parking Manager
class ParkingManager: ObservableObject {
    @Published var parkingMeters: [CustomMapAnnotation] = []
    @Published var userSearches: [CLLocationCoordinate2D] = []
    @Published var selectedMeter: CustomMapAnnotation?
    @Published var nearestMeter: (CustomMapAnnotation, Double)?
    @Published var errorMessage: String?

    func initializeMeters() {
        parkingMeters = [
            CustomMapAnnotation(latitude: 42.6334, longitude: -71.3162, status: "available"),
            CustomMapAnnotation(latitude: 42.6385, longitude: -71.3169, status: "occupied"),
            CustomMapAnnotation(latitude: 42.6400, longitude: -71.3180, status: "available"),
            CustomMapAnnotation(latitude: 42.6475, longitude: -71.3215, status: "available"),
            CustomMapAnnotation(latitude: 42.6430, longitude: -71.3120, status: "occupied")
        ]
    }

    func addUserSearch(at coordinate: CLLocationCoordinate2D) {
        userSearches.append(coordinate)
    }

    func findNearestGreenMeter(to location: CLLocationCoordinate2D) {
        let availableMeters = parkingMeters.filter { $0.status == "available" }
        guard !availableMeters.isEmpty else {
            errorMessage = "No available green meters found."
            nearestMeter = nil
            return
        }

        var closestMeter: CustomMapAnnotation?
        var smallestDistance = Double.greatestFiniteMagnitude

        for meter in availableMeters {
            let distance = haversineDistance(
                lat1: location.latitude,
                lon1: location.longitude,
                lat2: meter.coordinate.latitude,
                lon2: meter.coordinate.longitude
            )
            if distance < smallestDistance {
                smallestDistance = distance
                closestMeter = meter
            }
        }

        if let closest = closestMeter {
            nearestMeter = (closest, smallestDistance)
        } else {
            errorMessage = "No green meters within range."
            nearestMeter = nil
        }
    }
}

// MARK: - Location Search ViewModel
class LocationSearch: ObservableObject {
    @Published var region: MKCoordinateRegion
    @Published var searchText: String = ""

    init() {
        self.region = MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 42.6334, longitude: -71.3162),
            span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)
        )
    }

    func setRegion(to coordinate: CLLocationCoordinate2D) {
        region.center = coordinate
    }

    func zoom(by factor: Double) {
        region.span.latitudeDelta *= factor
        region.span.longitudeDelta *= factor
    }
}

// MARK: - Content View
struct ContentView: View {
    @StateObject private var parkingManager = ParkingManager()
    @ObservedObject private var locationSearch = LocationSearch()
    @State private var userLocation: CLLocationCoordinate2D?

    var body: some View {
        NavigationView {
            VStack {
                Text("TelePark")
                    .font(.largeTitle)
                    .fontWeight(.bold)
                    .padding()

                Divider()

                // Map with Annotations
                Map(
                    coordinateRegion: $locationSearch.region,
                    interactionModes: [.all],
                    annotationItems: parkingManager.parkingMeters
                ) { meter in
                    MapAnnotation(coordinate: meter.coordinate) {
                        MeterView(meter: meter)
                            .onTapGesture {
                                parkingManager.selectedMeter = meter
                            }
                    }
                }
                .frame(height: 300)
                .onAppear {
                    parkingManager.initializeMeters()
                }

                // Search Bar
                TextField("Search for location (lat, lon)", text: $locationSearch.searchText, onCommit: {
                    handleSearch()
                })
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding()

                Spacer()

                // Actions
                HStack {
                    Button("Reset Searches") {
                        parkingManager.userSearches.removeAll()
                        userLocation = nil
                    }
                    .padding()

                    Button("Initialize Meters") {
                        parkingManager.initializeMeters()
                    }
                    .padding()
                }

                // Display Nearest Green Meter
                if let nearest = parkingManager.nearestMeter {
                    VStack(alignment: .leading) {
                        Text("Nearest Green Meter:")
                            .font(.headline)
                        Text("Latitude: \(nearest.0.coordinate.latitude)")
                        Text("Longitude: \(nearest.0.coordinate.longitude)")
                        Text(String(format: "Distance: %.2f km", nearest.1))
                    }
                    .padding()
                }

                // Display Selected Meter Info
                if let selected = parkingManager.selectedMeter {
                    VStack(alignment: .leading) {
                        Text("Selected Meter:")
                            .font(.headline)
                        Text("Latitude: \(selected.coordinate.latitude)")
                        Text("Longitude: \(selected.coordinate.longitude)")
                        Text("Status: \(selected.status)")
                    }
                    .padding()
                }

                // Display Errors
                if let error = parkingManager.errorMessage {
                    Text(error)
                        .foregroundColor(.red)
                        .padding()
                }
            }
        }
    }

    private func handleSearch() {
        let components = locationSearch.searchText.split(separator: ",")
        if components.count == 2,
           let lat = Double(components[0]),
           let lon = Double(components[1]) {
            userLocation = CLLocationCoordinate2D(latitude: lat, longitude: lon)
            parkingManager.addUserSearch(at: userLocation!)
            parkingManager.findNearestGreenMeter(to: userLocation!)
        } else {
            parkingManager.errorMessage = "Invalid input. Please use the format 'lat, lon'."
        }
    }
}

// MARK: - Meter View
struct MeterView: View {
    let meter: CustomMapAnnotation

    var body: some View {
        VStack {
            Circle()
                .fill(meter.status == "available" ? Color.green : meter.status == "occupied" ? Color.red : Color.gray)
                .frame(width: 12, height: 12)
            Text(meter.status.capitalized)
                .font(.caption)
                .padding(4)
                .background(Color.white)
                .cornerRadius(5)
        }
    }
}

// MARK: - Preview
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
