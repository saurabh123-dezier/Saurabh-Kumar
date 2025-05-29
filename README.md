# To-Do
import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, query, onSnapshot, addDoc, updateDoc, deleteDoc, doc, Timestamp } from 'firebase/firestore';

// Main App component
function App() {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [todos, setTodos] = useState([]);
  const [newTask, setNewTask] = useState('');
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [darkMode, setDarkMode] = useState(false); // State for dark mode

  // Initialize Firebase and set up authentication
  useEffect(() => {
    try {
      // Retrieve Firebase config and app ID from global variables
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
      const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};

      // Initialize Firebase app
      const app = initializeApp(firebaseConfig);
      const firestoreDb = getFirestore(app);
      const firebaseAuth = getAuth(app);

      setDb(firestoreDb);
      setAuth(firebaseAuth);

      // Sign in with custom token if available, otherwise anonymously
      const signIn = async () => {
        try {
          if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
            await signInWithCustomToken(firebaseAuth, __initial_auth_token);
          } else {
            await signInAnonymously(firebaseAuth);
          }
        } catch (e) {
          console.error("Firebase authentication error:", e);
          setError("Authentication failed. Please try again.");
        }
      };

      // Listen for auth state changes
      const unsubscribeAuth = onAuthStateChanged(firebaseAuth, (user) => {
        if (user) {
          setUserId(user.uid);
          setLoading(false);
        } else {
          // If no user, sign in
          signIn();
        }
      });

      return () => unsubscribeAuth(); // Cleanup auth listener
    } catch (e) {
      console.error("Firebase initialization error:", e);
      setError("Failed to initialize the app. Please check console for details.");
      setLoading(false);
    }
  }, []);

  // Fetch todos from Firestore when db or userId changes
  useEffect(() => {
    if (db && userId) {
      const collectionPath = `artifacts/${__app_id}/public/data/todos`; // Public data for employees
      // Removed orderBy from Firestore query to avoid index requirement
      const q = query(collection(db, collectionPath));

      // Set up real-time listener for todos
      const unsubscribe = onSnapshot(q, (snapshot) => {
        const todosData = snapshot.docs.map(doc => ({
          id: doc.id,
          ...doc.data()
        }));

        // Sort data in memory after fetching to avoid Firestore index error
        const sortedTodos = todosData.sort((a, b) => {
          // Incomplete tasks first (false before true)
          if (a.completed !== b.completed) {
            return a.completed ? 1 : -1;
          }
          // Then by createdAt in descending order
          return (b.createdAt?.toMillis() || 0) - (a.createdAt?.toMillis() || 0);
        });

        setTodos(sortedTodos);
        setLoading(false);
      }, (err) => {
        console.error("Error fetching todos:", err);
        setError("Failed to load tasks. Please try again.");
        setLoading(false);
      });

      return () => unsubscribe(); // Cleanup snapshot listener
    }
  }, [db, userId]); // Re-run when db or userId changes

  // Add a new todo item
  const addTodo = async () => {
    if (newTask.trim() === '') {
      return; // Don't add empty tasks
    }
    if (!db || !userId) {
      setError("App not ready. Please wait or refresh.");
      return;
    }

    try {
      const collectionPath = `artifacts/${__app_id}/public/data/todos`;
      await addDoc(collection(db, collectionPath), {
        text: newTask,
        completed: false,
        createdAt: Timestamp.now(),
        userId: userId, // Store the ID of the user who created the task
        progress: 0, // Initialize progress to 0%
      });
      setNewTask(''); // Clear input field
    } catch (e) {
      console.error("Error adding document: ", e);
      setError("Failed to add task. Please try again.");
    }
  };

  // Toggle todo completion status
  const toggleTodo = async (id, completed) => {
    if (!db) {
      setError("App not ready. Please wait or refresh.");
      return;
    }
    try {
      const docRef = doc(db, `artifacts/${__app_id}/public/data/todos`, id);
      const newCompletedStatus = !completed;
      const newProgress = newCompletedStatus ? 100 : 0; // Set to 100 if completed, 0 if not

      await updateDoc(docRef, {
        completed: newCompletedStatus,
        progress: newProgress, // Update progress based on completion status
      });
    } catch (e) {
      console.error("Error updating document: ", e);
      setError("Failed to update task. Please try again.");
    }
  };

  // Update progress of a todo item
  const updateProgress = async (id, newProgressValue) => {
    if (!db) {
      setError("App not ready. Please wait or refresh.");
      return;
    }
    try {
      const docRef = doc(db, `artifacts/${__app_id}/public/data/todos`, id);
      // Ensure progress is within 0-100 range and is an integer
      const validatedProgress = Math.max(0, Math.min(100, parseInt(newProgressValue, 10)));

      await updateDoc(docRef, {
        progress: validatedProgress,
        // If progress is 100, mark as completed. If less than 100, mark as not completed.
        completed: validatedProgress === 100,
      });
    } catch (e) {
      console.error("Error updating progress: ", e);
      setError("Failed to update task progress. Please try again.");
    }
  };

  // Delete a todo item
  const deleteTodo = async (id) => {
    if (!db) {
      setError("App not ready. Please wait or refresh.");
      return;
    }
    try {
      const docRef = doc(db, `artifacts/${__app_id}/public/data/todos`, id);
      await deleteDoc(docRef);
    } catch (e) {
      console.error("Error deleting document: ", e);
      setError("Failed to delete task. Please try again.");
    }
  };

  // Toggle dark mode
  const toggleDarkMode = () => {
    setDarkMode(prevMode => !prevMode);
  };

  // Filter tasks into incomplete and completed
  const incompleteTodos = todos.filter(todo => !todo.completed);
  const completedTodos = todos.filter(todo => todo.completed);

  if (loading) {
    return (
      <div className="flex items-center justify-center min-h-screen bg-gray-100">
        <div className="text-lg font-semibold text-gray-700">Loading To-Do App...</div>
      </div>
    );
  }

  return (
    <div className={`min-h-screen ${darkMode ? 'bg-gray-900 text-gray-100' : 'bg-gradient-to-br from-red-100 via-yellow-100 to-blue-100 text-gray-800'} py-10 px-4 sm:px-6 lg:px-8 font-sans transition-colors duration-300`}>
      <script src="https://cdn.tailwindcss.com"></script>
      <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet" />

      <style>
        {`
          body {
            font-family: 'Inter', sans-serif;
          }
          /* Custom styling for range input thumb */
          input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 16px;
            height: 16px;
            background-color: ${darkMode ? '#60A5FA' : '#3B82F6'}; /* Blue-400 in dark, Blue-600 in light */
            border-radius: 9999px; /* Full rounded */
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
            cursor: pointer;
            margin-top: -6px; /* Adjust to center vertically */
          }
          input[type="range"]::-moz-range-thumb {
            width: 16px;
            height: 16px;
            background-color: ${darkMode ? '#60A5FA' : '#3B82F6'}; /* Blue-400 in dark, Blue-600 in light */
            border-radius: 9999px; /* Full rounded */
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
            cursor: pointer;
          }
          input[type="range"]::-webkit-slider-runnable-track {
            background: ${darkMode ? '#4B5563' : '#D1D5DB'}; /* Gray-600 in dark, Gray-300 in light */
            border-radius: 9999px;
            height: 4px;
          }
          input[type="range"]::-moz-range-track {
            background: ${darkMode ? '#4B5563' : '#D1D5DB'}; /* Gray-600 in dark, Gray-300 in light */
            border-radius: 9999px;
            height: 4px;
          }
        `}
      </style>

      <div className={`max-w-xl mx-auto rounded-xl shadow-2xl p-6 sm:p-8 border ${darkMode ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'}`}>
        <div className="flex justify-between items-center mb-8">
          <h1 className={`text-4xl font-extrabold tracking-tight ${darkMode ? 'text-gray-100' : 'text-gray-800'}`}>
            कर्मचारी टू-डू ऐप
          </h1>
          {/* Dark Mode Toggle Button */}
          <button
            onClick={toggleDarkMode}
            className={`relative inline-flex h-8 w-14 items-center rounded-full transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 ${darkMode ? 'bg-blue-600 focus:ring-blue-500' : 'bg-gray-200 focus:ring-blue-400'}`}
            aria-label="Toggle dark mode"
          >
            <span
              className={`inline-block h-6 w-6 transform rounded-full bg-white transition-transform duration-300 ${darkMode ? 'translate-x-7' : 'translate-x-1'}`}
            >
              {darkMode ? (
                <svg className="h-6 w-6 text-yellow-400" fill="currentColor" viewBox="0 0 24 24">
                  <path d="M12 3a6 6 0 009 5.86V12h.01L23 12.01A10 10 0 0112 22 10 10 0 012 12 10 10 0 0112 2zm0 1.99a8 8 0 00-7.99 7.99H12V4.99z" />
                </svg>
              ) : (
                <svg className="h-6 w-6 text-yellow-500" fill="currentColor" viewBox="0 0 24 24">
                  <path d="M12 2.25a.75.75 0 01.75.75v2.25a.75.75 0 01-1.5 0V3a.75.75 0 01.75-.75zM7.47 6.47a.75.75 0 011.06 0L9.5 7.44a.75.75 0 01-1.06 1.06l-1-1a.75.75 0 010-1.06zM4.5 12a.75.75 0 01.75-.75h2.25a.75.75 0 010 1.5H5.25a.75.75 0 01-.75-.75zM6.47 16.53a.75.75 0 010-1.06l1-1a.75.75 0 011.06 1.06l-1 1a.75.75 0 01-1.06 0zM12 17.25a.75.75 0 01-.75-.75v-2.25a.75.75 0 011.5 0v2.25a.75.75 0 01-.75.75zM16.53 16.53a.75.75 0 01-1.06 0l-1-1a.75.75 0 011.06-1.06l1 1a.75.75 0 010 1.06zM19.5 12a.75.75 0 01-.75-.75h-2.25a.75.75 0 010 1.5h2.25a.75.75 0 01.75-.75zM16.53 7.47a.75.75 0 010-1.06l1-1a.75.75 0 011.06 1.06l-1 1a.75.75 0 01-1.06 0z" />
                </svg>
              )}
            </span>
          </button>
        </div>


        {error && (
          <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-lg relative mb-6" role="alert">
            <strong className="font-bold">Error!</strong>
            <span className="block sm:inline ml-2">{error}</span>
          </div>
        )}

        {userId && (
          <p className={`text-sm text-center mb-6 ${darkMode ? 'text-gray-300' : 'text-gray-600'}`}>
            आपका यूजर ID: <span className={`font-mono px-2 py-1 rounded-md break-all ${darkMode ? 'bg-gray-700 text-gray-200' : 'bg-gray-100 text-gray-700'}`}>{userId}</span>
          </p>
        )}

        <div className="flex flex-col sm:flex-row gap-4 mb-8">
          <input
            type="text"
            className={`flex-grow p-3 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 shadow-sm ${darkMode ? 'bg-gray-700 border-gray-600 text-gray-100 placeholder-gray-400' : 'bg-white border-gray-300 text-gray-800 placeholder-gray-400'}`}
            placeholder="नया कार्य जोड़ें..."
            value={newTask}
            onChange={(e) => setNewTask(e.target.value)}
            onKeyPress={(e) => {
              if (e.key === 'Enter') {
                addTodo();
              }
            }}
          />
          <button
            onClick={addTodo}
            className="px-6 py-3 bg-blue-600 text-white font-semibold rounded-lg shadow-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-offset-2 transition duration-300 ease-in-out transform hover:scale-105"
          >
            कार्य जोड़ें
          </button>
        </div>

        {/* Incomplete Tasks Section */}
        <div className="mb-8">
          <h2 className={`text-2xl font-bold mb-4 border-b-2 pb-2 ${darkMode ? 'text-gray-200 border-blue-500' : 'text-gray-700 border-blue-400'}`}>अधूरे कार्य</h2>
          {incompleteTodos.length === 0 ? (
            <p className={`text-center text-lg py-5 ${darkMode ? 'text-gray-400' : 'text-gray-500'}`}>कोई अधूरा कार्य नहीं है।</p>
          ) : (
            <ul className="space-y-4">
              {incompleteTodos.map((todo) => (
                <li
                  key={todo.id}
                  className={`flex flex-col sm:flex-row items-start sm:items-center justify-between p-4 rounded-lg shadow-sm border transition duration-200 ease-in-out ${darkMode ? 'bg-gray-700 border-gray-600 hover:bg-gray-600' : 'bg-gray-50 border-gray-100 hover:bg-gray-100'}`}
                >
                  {/* Task Text and Checkbox */}
                  <div className="flex items-center flex-grow min-w-0 mb-3 sm:mb-0">
                    <input
                      type="checkbox"
                      checked={todo.completed}
                      onChange={() => toggleTodo(todo.id, todo.completed)}
                      className="form-checkbox h-5 w-5 text-blue-600 rounded-md focus:ring-blue-500 cursor-pointer flex-shrink-0"
                    />
                    <span
                      className={`ml-4 text-lg break-words ${todo.completed ? 'line-through text-gray-500' : (darkMode ? 'text-gray-100' : 'text-gray-800')}`}
                    >
                      {todo.text}
                    </span>
                  </div>

                  {/* Progress Bar, Slider, and Delete Button */}
                  <div className="flex items-center ml-0 sm:ml-4 w-full sm:w-48 flex-shrink-0">
                    <div className="flex-grow mr-2">
                      {/* Progress Bar Display */}
                      <div className={`w-full rounded-full h-2.5 mb-1 relative overflow-hidden ${darkMode ? 'bg-gray-600' : 'bg-gray-200'}`}>
                        <div
                          className="bg-green-500 h-2.5 rounded-full transition-all duration-300 ease-in-out"
                          style={{ width: `${todo.progress || 0}%` }} // Ensure default to 0 if undefined
                        ></div>
                        <span className={`absolute top-0 left-0 right-0 bottom-0 flex items-center justify-center text-xs font-semibold ${darkMode ? 'text-gray-200' : 'text-gray-800'}`} style={{ textShadow: `0 0 2px ${darkMode ? 'black' : 'white'}` }}>
                          {todo.progress || 0}%
                        </span>
                      </div>
                      {/* Progress Slider Input */}
                      <input
                        type="range"
                        min="0"
                        max="100"
                        value={todo.progress || 0} // Ensure default to 0
                        onChange={(e) => updateProgress(todo.id, e.target.value)}
                        className={`w-full h-4 rounded-lg appearance-none cursor-pointer ${darkMode ? 'bg-gray-600' : 'bg-gray-300'}`}
                      />
                    </div>
                    {/* Delete Button */}
                    <button
                      onClick={() => deleteTodo(todo.id)}
                      className="p-2 bg-red-500 text-white rounded-full shadow-md hover:bg-red-600 focus:outline-none focus:ring-2 focus:ring-red-400 focus:ring-offset-2 transition duration-300 ease-in-out transform hover:scale-110 flex-shrink-0"
                      aria-label="Delete task"
                    >
                      <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path>
                      </svg>
                    </button>
                  </div>
                </li>
              ))}
            </ul>
          )}
        </div>

        {/* Completed Tasks Section */}
        <div>
          <h2 className={`text-2xl font-bold mb-4 border-b-2 pb-2 ${darkMode ? 'text-gray-200 border-green-500' : 'text-gray-700 border-green-400'}`}>पूरे हुए कार्य</h2>
          {completedTodos.length === 0 ? (
            <p className={`text-center text-lg py-5 ${darkMode ? 'text-gray-400' : 'text-gray-500'}`}>अभी तक कोई कार्य पूरा नहीं हुआ है।</p>
          ) : (
            <ul className="space-y-4">
              {completedTodos.map((todo) => (
                <li
                  key={todo.id}
                  className={`flex flex-col sm:flex-row items-start sm:items-center justify-between p-4 rounded-lg shadow-sm border transition duration-200 ease-in-out ${darkMode ? 'bg-gray-700 border-gray-600 hover:bg-gray-600' : 'bg-gray-50 border-gray-100 hover:bg-gray-100'}`}
                >
                  {/* Task Text and Checkbox */}
                  <div className="flex items-center flex-grow min-w-0 mb-3 sm:mb-0">
                    <input
                      type="checkbox"
                      checked={todo.completed}
                      onChange={() => toggleTodo(todo.id, todo.completed)}
                      className="form-checkbox h-5 w-5 text-blue-600 rounded-md focus:ring-blue-500 cursor-pointer flex-shrink-0"
                    />
                    <span
                      className={`ml-4 text-lg break-words ${todo.completed ? 'line-through text-gray-500' : (darkMode ? 'text-gray-100' : 'text-gray-800')}`}
                    >
                      {todo.text}
                    </span>
                  </div>

                  {/* Progress Bar, Slider, and Delete Button */}
                  <div className="flex items-center ml-0 sm:ml-4 w-full sm:w-48 flex-shrink-0">
                    <div className="flex-grow mr-2">
                      {/* Progress Bar Display */}
                      <div className={`w-full rounded-full h-2.5 mb-1 relative overflow-hidden ${darkMode ? 'bg-gray-600' : 'bg-gray-200'}`}>
                        <div
                          className="bg-green-500 h-2.5 rounded-full transition-all duration-300 ease-in-out"
                          style={{ width: `${todo.progress || 0}%` }} // Ensure default to 0 if undefined
                        ></div>
                        <span className={`absolute top-0 left-0 right-0 bottom-0 flex items-center justify-center text-xs font-semibold ${darkMode ? 'text-gray-200' : 'text-gray-800'}`} style={{ textShadow: `0 0 2px ${darkMode ? 'black' : 'white'}` }}>
                          {todo.progress || 0}%
                        </span>
                      </div>
                      {/* Progress Slider Input */}
                      <input
                        type="range"
                        min="0"
                        max="100"
                        value={todo.progress || 0} // Ensure default to 0
                        onChange={(e) => updateProgress(todo.id, e.target.value)}
                        className={`w-full h-4 rounded-lg appearance-none cursor-pointer ${darkMode ? 'bg-gray-600' : 'bg-gray-300'}`}
                      />
                    </div>
                    {/* Delete Button */}
                    <button
                      onClick={() => deleteTodo(todo.id)}
                      className="p-2 bg-red-500 text-white rounded-full shadow-md hover:bg-red-600 focus:outline-none focus:ring-2 focus:ring-red-400 focus:ring-offset-2 transition duration-300 ease-in-out transform hover:scale-110 flex-shrink-0"
                      aria-label="Delete task"
                    >
                      <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path>
                      </svg>
                    </button>
                  </div>
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>
    </div>
  );
}

export default App;
