
import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, getDoc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, where, getDocs, arrayUnion, arrayRemove } from 'firebase/firestore';

// Context for Firebase and User
const AppContext = createContext(null);

// Main App Component
const App = () => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [view, setView] = useState('user'); // 'user' or 'admin'
    const [message, setMessage] = useState('');
    const [showModal, setShowModal] = useState(false);

    // Global variables for Firebase config and app ID (provided by the environment)
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
    const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

    useEffect(() => {
        // Initialize Firebase services
        try {
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const firebaseAuth = getAuth(app);
            setDb(firestore);
            setAuth(firebaseAuth);

            // Listen for authentication state changes
            const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    // Sign in anonymously if no token is provided or user logs out
                    try {
                        if (initialAuthToken) {
                            await signInWithCustomToken(firebaseAuth, initialAuthToken);
                        } else {
                            await signInAnonymously(firebaseAuth);
                        }
                        setUserId(firebaseAuth.currentUser?.uid || crypto.randomUUID()); // Fallback to random UUID if sign-in fails
                    } catch (error) {
                        console.error("Error signing in:", error);
                        setUserId(crypto.randomUUID()); // Use a random UUID if anonymous sign-in also fails
                    }
                }
                setIsAuthReady(true); // Mark auth as ready after initial check/sign-in
            });

            return () => unsubscribe(); // Cleanup auth listener on unmount
        } catch (error) {
            console.error("Failed to initialize Firebase:", error);
            setMessage("Failed to initialize the application. Please try again later.");
            setShowModal(true);
        }
    }, [appId, firebaseConfig, initialAuthToken]); // Dependencies for useEffect

    // Function to show a message modal
    const showMessage = (msg) => {
        setMessage(msg);
        setShowModal(true);
    };

    // Modal component for messages
    const MessageModal = ({ message, onClose }) => {
        if (!showModal) return null;
        return (
            <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex justify-center items-center z-50">
                <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full text-center">
                    <p className="text-lg font-semibold mb-4">{message}</p>
                    <button
                        onClick={onClose}
                        className="px-6 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition duration-200"
                    >
                        Close
                    </button>
                </div>
            </div>
        );
    };

    if (!isAuthReady) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100">
                <div className="text-lg font-semibold text-gray-700">Loading application...</div>
            </div>
        );
    }

    return (
        <AppContext.Provider value={{ db, auth, userId, appId, showMessage }}>
            <div className="min-h-screen bg-gray-100 font-sans antialiased flex flex-col">
                <header className="bg-blue-800 text-white p-4 shadow-md">
                    <div className="container mx-auto flex justify-between items-center">
                        <h1 className="text-3xl font-bold rounded-md px-3 py-1 bg-blue-700">Eazyvenue</h1>
                        <div className="flex space-x-4">
                            <button
                                onClick={() => setView('user')}
                                className={`px-4 py-2 rounded-md transition duration-200 ${view === 'user' ? 'bg-blue-600 shadow-lg' : 'hover:bg-blue-700'}`}
                            >
                                User View
                            </button>
                            <button
                                onClick={() => setView('admin')}
                                className={`px-4 py-2 rounded-md transition duration-200 ${view === 'admin' ? 'bg-blue-600 shadow-lg' : 'hover:bg-blue-700'}`}
                            >
                                Admin View
                            </button>
                        </div>
                    </div>
                </header>

                <main className="flex-grow container mx-auto p-6">
                    {userId && (
                        <div className="mb-6 p-4 bg-blue-100 text-blue-800 rounded-lg shadow-sm">
                            <p className="text-sm font-medium">Your User ID: <span className="font-mono break-all">{userId}</span></p>
                        </div>
                    )}
                    {view === 'user' ? <UserDashboard /> : <AdminDashboard />}
                </main>

                <MessageModal message={message} onClose={() => setShowModal(false)} />
            </div>
        </AppContext.Provider>
    );
};

// Admin Dashboard Component
const AdminDashboard = () => {
    const { db, userId, appId, showMessage } = useContext(AppContext);
    const [venues, setVenues] = useState([]);
    const [newVenue, setNewVenue] = useState({ name: '', location: '', description: '', imageUrl: 'https://placehold.co/400x200/ADD8E6/000000?text=Venue' });
    const [selectedVenueId, setSelectedVenueId] = useState(null);
    const [blockDate, setBlockDate] = useState('');

    // Fetch venues owned by the current user
    useEffect(() => {
        if (!db || !userId) return;

        const venuesCollectionRef = collection(db, `artifacts/${appId}/public/data/venues`);
        const q = query(venuesCollectionRef, where("ownerId", "==", userId));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const fetchedVenues = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setVenues(fetchedVenues);
        }, (error) => {
            console.error("Error fetching admin venues:", error);
            showMessage("Error fetching your venues.");
        });

        return () => unsubscribe();
    }, [db, userId, appId, showMessage]);

    // Handle adding a new venue
    const handleAddVenue = async (e) => {
        e.preventDefault();
        if (!db || !userId) {
            showMessage("Database not ready or user not authenticated.");
            return;
        }
        if (!newVenue.name || !newVenue.location || !newVenue.description) {
            showMessage("Please fill in all venue details.");
            return;
        }

        try {
            await addDoc(collection(db, `artifacts/${appId}/public/data/venues`), {
                ...newVenue,
                ownerId: userId,
                unavailableDates: [],
                createdAt: new Date().toISOString(),
            });
            setNewVenue({ name: '', location: '', description: '', imageUrl: 'https://placehold.co/400x200/ADD8E6/000000?text=Venue' });
            showMessage("Venue added successfully!");
        } catch (error) {
            console.error("Error adding venue:", error);
            showMessage("Failed to add venue.");
        }
    };

    // Handle blocking a date for a venue
    const handleBlockDate = async () => {
        if (!db || !selectedVenueId || !blockDate) {
            showMessage("Please select a venue and a date to block.");
            return;
        }

        try {
            const venueRef = doc(db, `artifacts/${appId}/public/data/venues`, selectedVenueId);
            await updateDoc(venueRef, {
                unavailableDates: arrayUnion(blockDate) // Add date to array
            });
            setBlockDate('');
            showMessage(`Date ${blockDate} blocked for venue.`);
        } catch (error) {
            console.error("Error blocking date:", error);
            showMessage("Failed to block date.");
        }
    };

    // Handle unblocking a date for a venue
    const handleUnblockDate = async (dateToUnblock) => {
        if (!db || !selectedVenueId || !dateToUnblock) {
            showMessage("No venue or date selected to unblock.");
            return;
        }

        try {
            const venueRef = doc(db, `artifacts/${appId}/public/data/venues`, selectedVenueId);
            await updateDoc(venueRef, {
                unavailableDates: arrayRemove(dateToUnblock) // Remove date from array
            });
            showMessage(`Date ${dateToUnblock} unblocked for venue.`);
        } catch (error) {
            console.error("Error unblocking date:", error);
            showMessage("Failed to unblock date.");
        }
    };

    return (
        <div className="bg-white p-8 rounded-lg shadow-lg">
            <h2 className="text-3xl font-bold text-gray-800 mb-6 border-b-2 pb-2 border-blue-200">Admin Dashboard</h2>

            {/* Add New Venue Form */}
            <section className="mb-10 p-6 bg-blue-50 rounded-lg shadow-md">
                <h3 className="text-2xl font-semibold text-gray-700 mb-4">Add New Venue</h3>
                <form onSubmit={handleAddVenue} className="space-y-4">
                    <div>
                        <label htmlFor="venueName" className="block text-gray-700 text-sm font-bold mb-2">Venue Name:</label>
                        <input
                            type="text"
                            id="venueName"
                            value={newVenue.name}
                            onChange={(e) => setNewVenue({ ...newVenue, name: e.target.value })}
                            className="shadow appearance-none border rounded-md w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400"
                            placeholder="e.g., Grand Ballroom"
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="venueLocation" className="block text-gray-700 text-sm font-bold mb-2">Location:</label>
                        <input
                            type="text"
                            id="venueLocation"
                            value={newVenue.location}
                            onChange={(e) => setNewVenue({ ...newVenue, location: e.target.value })}
                            className="shadow appearance-none border rounded-md w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400"
                            placeholder="e.g., New York, NY"
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="venueDescription" className="block text-gray-700 text-sm font-bold mb-2">Description:</label>
                        <textarea
                            id="venueDescription"
                            value={newVenue.description}
                            onChange={(e) => setNewVenue({ ...newVenue, description: e.target.value })}
                            className="shadow appearance-none border rounded-md w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400 h-24"
                            placeholder="A brief description of the venue..."
                            required
                        ></textarea>
                    </div>
                    <div>
                        <label htmlFor="venueImage" className="block text-gray-700 text-sm font-bold mb-2">Image URL (Optional):</label>
                        <input
                            type="text"
                            id="venueImage"
                            value={newVenue.imageUrl}
                            onChange={(e) => setNewVenue({ ...newVenue, imageUrl: e.target.value })}
                            className="shadow appearance-none border rounded-md w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400"
                            placeholder="e.g., https://example.com/image.jpg"
                        />
                    </div>
                    <button
                        type="submit"
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-6 rounded-md focus:outline-none focus:shadow-outline transition duration-200 shadow-md"
                    >
                        Add Venue
                    </button>
                </form>
            </section>

            {/* Manage Venues Section */}
            <section className="p-6 bg-blue-50 rounded-lg shadow-md">
                <h3 className="text-2xl font-semibold text-gray-700 mb-4">Your Venues</h3>
                {venues.length === 0 ? (
                    <p className="text-gray-600">You haven't added any venues yet. Add one above!</p>
                ) : (
                    <div className="space-y-6">
                        {venues.map((venue) => (
                            <div key={venue.id} className="bg-white p-5 rounded-lg shadow-sm border border-blue-100">
                                <h4 className="text-xl font-bold text-gray-800 mb-2">{venue.name}</h4>
                                <p className="text-gray-600 mb-2">{venue.location}</p>
                                <p className="text-gray-700 text-sm mb-4">{venue.description}</p>
                                <img src={venue.imageUrl} alt={venue.name} className="w-full h-48 object-cover rounded-md mb-4" onError={(e) => { e.target.onerror = null; e.target.src = "https://placehold.co/400x200/ADD8E6/000000?text=Image+Error"; }} />

                                {/* Block/Unblock Dates */}
                                <div className="mt-4 border-t pt-4 border-blue-100">
                                    <h5 className="text-lg font-semibold text-gray-700 mb-3">Manage Availability</h5>
                                    <div className="flex items-center space-x-3 mb-4">
                                        <input
                                            type="date"
                                            value={selectedVenueId === venue.id ? blockDate : ''}
                                            onChange={(e) => {
                                                setSelectedVenueId(venue.id);
                                                setBlockDate(e.target.value);
                                            }}
                                            className="shadow appearance-none border rounded-md py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400"
                                        />
                                        <button
                                            onClick={handleBlockDate}
                                            className="bg-red-500 hover:bg-red-600 text-white font-bold py-2 px-4 rounded-md transition duration-200 shadow-sm"
                                        >
                                            Block Date
                                        </button>
                                    </div>
                                    {venue.unavailableDates && venue.unavailableDates.length > 0 && (
                                        <div className="mt-3">
                                            <p className="text-sm font-medium text-gray-700 mb-2">Blocked Dates:</p>
                                            <div className="flex flex-wrap gap-2">
                                                {venue.unavailableDates.map((date) => (
                                                    <span key={date} className="inline-flex items-center bg-gray-200 text-gray-800 text-xs font-semibold px-2.5 py-0.5 rounded-full">
                                                        {date}
                                                        <button
                                                            onClick={() => {
                                                                setSelectedVenueId(venue.id);
                                                                handleUnblockDate(date);
                                                            }}
                                                            className="ml-1 text-red-600 hover:text-red-800"
                                                        >
                                                            &times;
                                                        </button>
                                                    </span>
                                                ))}
                                            </div>
                                        </div>
                                    )}
                                </div>
                            </div>
                        ))}
                    </div>
                )}
            </section>
        </div>
    );
};

// User Dashboard Component
const UserDashboard = () => {
    const { db, userId, appId, showMessage } = useContext(AppContext);
    const [venues, setVenues] = useState([]);
    const [bookingDate, setBookingDate] = useState('');
    const [selectedVenueForBooking, setSelectedVenueForBooking] = useState(null);
    const [userBookings, setUserBookings] = useState([]);

    // Fetch all venues
    useEffect(() => {
        if (!db) return;

        const venuesCollectionRef = collection(db, `artifacts/${appId}/public/data/venues`);
        const unsubscribe = onSnapshot(venuesCollectionRef, (snapshot) => {
            const fetchedVenues = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setVenues(fetchedVenues);
        }, (error) => {
            console.error("Error fetching all venues:", error);
            showMessage("Error fetching venues for browsing.");
        });

        return () => unsubscribe();
    }, [db, appId, showMessage]);

    // Fetch user's bookings
    useEffect(() => {
        if (!db || !userId) return;

        const bookingsCollectionRef = collection(db, `artifacts/${appId}/public/data/bookings`);
        const q = query(bookingsCollectionRef, where("userId", "==", userId));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const fetchedBookings = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setUserBookings(fetchedBookings);
        }, (error) => {
            console.error("Error fetching user bookings:", error);
            showMessage("Error fetching your bookings.");
        });

        return () => unsubscribe();
    }, [db, userId, appId, showMessage]);


    // Handle booking a venue
    const handleBookVenue = async (venueId) => {
        if (!db || !userId || !bookingDate) {
            showMessage("Please select a date to book.");
            return;
        }

        const selectedVenue = venues.find(v => v.id === venueId);
        if (!selectedVenue) {
            showMessage("Venue not found.");
            return;
        }

        // Check if the date is unavailable
        if (selectedVenue.unavailableDates && selectedVenue.unavailableDates.includes(bookingDate)) {
            showMessage(`Venue is unavailable on ${bookingDate}. Please choose another date.`);
            return;
        }

        // Check for existing bookings for this venue on this date
        const bookingsRef = collection(db, `artifacts/${appId}/public/data/bookings`);
        const q = query(bookingsRef,
            where("venueId", "==", venueId),
            where("bookingDate", "==", bookingDate)
        );

        try {
            const querySnapshot = await getDocs(q);
            if (!querySnapshot.empty) {
                showMessage(`Venue is already booked by someone else on ${bookingDate}.`);
                return;
            }

            // Proceed with booking
            await addDoc(bookingsRef, {
                venueId: venueId,
                userId: userId,
                bookingDate: bookingDate,
                status: 'confirmed', // Or 'pending' if a confirmation step is needed
                bookedAt: new Date().toISOString(),
            });

            // Automatically update venue's unavailable dates
            const venueRef = doc(db, `artifacts/${appId}/public/data/venues`, venueId);
            await updateDoc(venueRef, {
                unavailableDates: arrayUnion(bookingDate)
            });

            showMessage(`Venue booked successfully for ${bookingDate}!`);
            setSelectedVenueForBooking(null); // Close booking form
            setBookingDate('');
        } catch (error) {
            console.error("Error booking venue:", error);
            showMessage("Failed to book venue.");
        }
    };

    return (
        <div className="bg-white p-8 rounded-lg shadow-lg">
            <h2 className="text-3xl font-bold text-gray-800 mb-6 border-b-2 pb-2 border-blue-200">User Dashboard</h2>

            {/* Browse Venues Section */}
            <section className="mb-10">
                <h3 className="text-2xl font-semibold text-gray-700 mb-4">Available Venues</h3>
                {venues.length === 0 ? (
                    <p className="text-gray-600">No venues available yet. Check back later!</p>
                ) : (
                    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                        {venues.map((venue) => (
                            <div key={venue.id} className="bg-white rounded-lg shadow-md overflow-hidden border border-blue-100 hover:shadow-xl transition-shadow duration-300">
                                <img
                                    src={venue.imageUrl}
                                    alt={venue.name}
                                    className="w-full h-48 object-cover"
                                    onError={(e) => { e.target.onerror = null; e.target.src = "https://placehold.co/400x200/ADD8E6/000000?text=Image+Error"; }}
                                />
                                <div className="p-5">
                                    <h4 className="text-xl font-bold text-gray-800 mb-2">{venue.name}</h4>
                                    <p className="text-gray-600 mb-2">{venue.location}</p>
                                    <p className="text-gray-700 text-sm mb-4 line-clamp-3">{venue.description}</p>

                                    {/* Availability Info */}
                                    {venue.unavailableDates && venue.unavailableDates.length > 0 && (
                                        <div className="mb-3">
                                            <p className="text-sm font-medium text-red-600">
                                                Unavailable on: {venue.unavailableDates.join(', ')}
                                            </p>
                                        </div>
                                    )}

                                    {/* Booking Form */}
                                    {selectedVenueForBooking === venue.id ? (
                                        <div className="mt-4 p-4 bg-blue-50 rounded-md">
                                            <h5 className="text-lg font-semibold text-gray-700 mb-3">Book This Venue</h5>
                                            <input
                                                type="date"
                                                value={bookingDate}
                                                onChange={(e) => setBookingDate(e.target.value)}
                                                className="shadow appearance-none border rounded-md w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400 mb-3"
                                            />
                                            <div className="flex space-x-2">
                                                <button
                                                    onClick={() => handleBookVenue(venue.id)}
                                                    className="flex-1 bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-md transition duration-200 shadow-sm"
                                                >
                                                    Confirm Booking
                                                </button>
                                                <button
                                                    onClick={() => setSelectedVenueForBooking(null)}
                                                    className="flex-1 bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-2 px-4 rounded-md transition duration-200 shadow-sm"
                                                >
                                                    Cancel
                                                </button>
                                            </div>
                                        </div>
                                    ) : (
                                        <button
                                            onClick={() => {
                                                setSelectedVenueForBooking(venue.id);
                                                setBookingDate(''); // Reset date when opening form
                                            }}
                                            className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline transition duration-200 shadow-md"
                                        >
                                            Book Now
                                        </button>
                                    )}
                                </div>
                            </div>
                        ))}
                    </div>
                )}
            </section>

            {/* Your Bookings Section */}
            <section className="p-6 bg-blue-50 rounded-lg shadow-md">
                <h3 className="text-2xl font-semibold text-gray-700 mb-4">Your Bookings</h3>
                {userBookings.length === 0 ? (
                    <p className="text-gray-600">You haven't made any bookings yet.</p>
                ) : (
                    <div className="space-y-4">
                        {userBookings.map((booking) => {
                            const bookedVenue = venues.find(v => v.id === booking.venueId);
                            return (
                                <div key={booking.id} className="bg-white p-4 rounded-lg shadow-sm border border-blue-100 flex items-center justify-between">
                                    <div>
                                        <p className="text-lg font-semibold text-gray-800">
                                            {bookedVenue ? bookedVenue.name : 'Unknown Venue'}
                                        </p>
                                        <p className="text-gray-600 text-sm">
                                            Date: <span className="font-medium">{booking.bookingDate}</span>
                                        </p>
                                        <p className="text-gray-600 text-sm">
                                            Status: <span className="font-medium text-green-700">{booking.status}</span>
                                        </p>
                                    </div>
                                    {/* Add a cancel booking button if desired */}
                                    {/* <button className="bg-red-500 hover:bg-red-600 text-white py-1 px-3 rounded-md text-sm">Cancel</button> */}
                                </div>
                            );
                        })}
                    </div>
                )}
            </section>
        </div>
    );
};

export default App;

