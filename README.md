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
    @Published var recentSearches: [String] = []
    @Published var mapView = MKMapView()

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

// MARK: - Parking Manager
class ParkingManager: ObservableObject {
    @Published var parkingMeters: [CustomMapAnnotation] = []
    @Published var isLoggedIn: Bool = false
    @Published var showLoginSheet: Bool = false

    func addMeter(at coordinate: CLLocationCoordinate2D) {
        let newMeter = CustomMapAnnotation(coordinate: coordinate, status: "available")
        parkingMeters.append(newMeter)
    }

    func updateMeterStatus(id: UUID, to status: String) {
        if let index = parkingMeters.firstIndex(where: { $0.id == id }) {
            parkingMeters[index].status = status
        }
    }

    func resetMeters() {
        parkingMeters.removeAll()
    }
}

// MARK: - Content View
struct ContentView: View {
    @StateObject private var parkingManager = ParkingManager()
    @State private var selectedMeter: CustomMapAnnotation?
    @ObservedObject private var locationSearch = LocationSearch()
    @State private var searchQuery: String = ""

    var body: some View {
        NavigationView {
            VStack {
                // App Header
                Text("TelePark")
                    .font(.largeTitle)
                    .fontWeight(.bold)
                    .padding()

                Divider()

                // Map View with Meters
                ZStack {
                    Map(
                        coordinateRegion: $locationSearch.region,
                        interactionModes: [.all],
                        annotationItems: parkingManager.parkingMeters
                    ) { meter in
                        MapAnnotation(coordinate: meter.coordinate) {
                            MeterView(meter: meter)
                                .onTapGesture {
                                    selectedMeter = meter
                                }
                        }
                    }
                    .frame(height: 300)
                }

                // Meter Management Buttons
                HStack {
                    Button("Add Meter") {
                        let centerCoordinate = locationSearch.region.center
                        parkingManager.addMeter(at: centerCoordinate)
                        print("Added meter at: \(centerCoordinate.latitude), \(centerCoordinate.longitude)")
                    }
                    .padding()
                    .buttonStyle(.borderedProminent)

                    Button("Reset Meters") {
                        parkingManager.resetMeters()
                        print("Reset all meters")
                    }
                    .padding()
                    .buttonStyle(.bordered)
                }

                // Zoom Controls
                HStack {
                    Button(action: {
                        locationSearch.zoom(by: 0.5)
                    }) {
                        Label("Zoom In", systemImage: "plus.magnifyingglass")
                    }
                    .padding()

                    Button(action: {
                        locationSearch.zoom(by: 2.0)
                    }) {
                        Label("Zoom Out", systemImage: "minus.magnifyingglass")
                    }
                    .padding()
                }

                Spacer()

                // Login/Logout Button
                Button(parkingManager.isLoggedIn ? "Logout" : "Login as Parking Director") {
                    parkingManager.showLoginSheet.toggle()
                }
                .padding()

                // Selected Meter Info
                if let selected = selectedMeter {
                    VStack(alignment: .leading) {
                        Text("Selected Meter:")
                            .font(.headline)
                        Text("Latitude: \(selected.coordinate.latitude)")
                        Text("Longitude: \(selected.coordinate.longitude)")
                        Text("Status: \(selected.status)")
                    }
                    .padding()
                }
            }
            .sheet(isPresented: $parkingManager.showLoginSheet) {
                VStack {
                    Text(parkingManager.isLoggedIn ? "Logout Successful" : "Login Sheet")
                        .font(.title)
                        .padding()
                    Button("Close") {
                        parkingManager.showLoginSheet = false
                    }
                    .padding()
                }
            }
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
                .frame(width: 10, height: 10)
            Text(meter.status.capitalized)
                .font(.caption)
                .padding(4)
                .background(Color.white)
                .cornerRadius(5)
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

// MARK: - Preview
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
