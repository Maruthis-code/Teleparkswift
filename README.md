import SwiftUI
import MapKit

public init(_ title: LocalizedStringKey, coordinate: CLLocationCoordinate2D, @ViewBuilder content: () -> some View)

// Protocol Definition
protocol MapAnnotationProtocol {
    var coordinate: CLLocationCoordinate2D { get }
}

// Custom Annotation Struct
struct CustomMapAnnotation: Identifiable, MapAnnotationProtocol {
    let id = UUID()
    let coordinate: CLLocationCoordinate2D // Conforms to the protocol requirement
    let status: String                     // Additional property for status
}

// ObservableObject for Location Search
class LocationSearch: ObservableObject {
    @Published var region = MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 42.6334, longitude: -71.3162),
        span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)
    )
    @Published var errorMessage: String?

    // Function to search for a location
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

// Main Content View
struct ContentView: View {
    @State private var searchQuery = "UMass Lowell"
    @StateObject var locationSearch = LocationSearch()
    @State private var parkingMeters: [CustomMapAnnotation] = [
        CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6335, longitude: -71.3161), status: "available"),
        CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6340, longitude: -71.3155), status: "occupied"),
        CustomMapAnnotation(coordinate: CLLocationCoordinate2D(latitude: 42.6329, longitude: -71.3170), status: "available")
    ]

    var body: some View {
        VStack {
            // Search Bar
            TextField("Search for a location", text: $searchQuery)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding()
                .onSubmit {
                    locationSearch.searchLocation(query: searchQuery)
                }

            // Map with Custom Annotations
            // Updated Map with Annotation API
            Map(coordinateRegion: $locationSearch.region, showsUserLocation: false, annotationItems: parkingMeters) { meter in
                Annotation("Parking Meter", coordinate: meter.coordinate) {
                    VStack {
                        Circle()
                            .fill(meter.status == "available" ? Color.green : Color.red)
                            .frame(width: 10, height: 10)
                            .accessibilityLabel("Parking meter with status \(meter.status)")

                        Text(meter.status)
                            .font(.caption)
                            .padding(4)
                            .background(Color.white)
                            .cornerRadius(5)
                            .foregroundColor(.black)
                    }
                }
            }
            .ignoresSafeArea()

            // Display Error Message
            if let errorMessage = locationSearch.errorMessage {
                Text(errorMessage)
                    .foregroundColor(.red)
                    .padding()
            }
        }
    }
}

