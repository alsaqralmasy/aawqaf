import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, query, orderBy, onSnapshot, addDoc, serverTimestamp, deleteDoc, doc, setDoc, updateDoc, arrayUnion } from 'firebase/firestore';

// Define a context for Firebase and user data
const AppContext = createContext(null);

// Custom hook to use the app context
const useAppContext = () => {
    const context = useContext(AppContext);
    if (!context) {
        throw new Error('useAppContext must be used within an AppProvider');
    }
    return context;
};

// Message Modal Component
const MessageModal = ({ message, type, onClose }) => {
    if (!message) return null;

    const bgColor = type === 'success' ? 'bg-green-100 border-green-400 text-green-700' : 'bg-red-100 border-red-400 text-red-700';
    const textColor = type === 'success' ? 'text-green-700' : 'text-red-700';

    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className={`relative ${bgColor} border rounded-lg shadow-xl p-6 max-w-sm w-full text-center`}>
                <p className={`font-bold text-lg mb-4 ${textColor}`}>{message}</p>
                <button
                    onClick={onClose}
                    className="mt-4 px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition duration-200 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-50"
                >
                    إغلاق
                </button>
            </div>
        </div>
    );
};


// App Provider Component to initialize Firebase and provide context
const AppProvider = ({ children }) => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [loading, setLoading] = useState(true);
    const [message, setMessage] = useState('');
    const [messageType, setMessageType] = useState('success');

    // Function to show messages
    const showMessage = (msg, type = 'success') => {
        setMessage(msg);
        setMessageType(type);
        setTimeout(() => setMessage(''), 3000); // Clear message after 3 seconds
    };

    useEffect(() => {
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};

        try {
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const firebaseAuth = getAuth(app);

            setDb(firestore);
            setAuth(firebaseAuth);

            // Listen for auth state changes
            const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                    console.log("Firebase Auth: User signed in with UID:", user.uid);
                } else {
                    // Sign in anonymously if no user is logged in
                    if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                        try {
                            await signInWithCustomToken(firebaseAuth, __initial_auth_token);
                            setUserId(firebaseAuth.currentUser.uid);
                            console.log("Firebase Auth: Signed in with custom token, UID:", firebaseAuth.currentUser.uid);
                        } catch (error) {
                            console.error("Error signing in with custom token:", error);
                            showMessage("خطأ في تسجيل الدخول الرمزي. جاري تسجيل الدخول كضيف.", "error");
                            await signInAnonymously(firebaseAuth);
                            setUserId(firebaseAuth.currentUser.uid);
                            console.log("Firebase Auth: Signed in anonymously after custom token failure, UID:", firebaseAuth.currentUser.uid);
                        }
                    } else {
                        await signInAnonymously(firebaseAuth);
                        setUserId(firebaseAuth.currentUser.uid);
                        console.log("Firebase Auth: Signed in anonymously, UID:", firebaseAuth.currentUser.uid);
                    }
                }
                setLoading(false);
            });

            return () => unsubscribe(); // Cleanup auth listener
        } catch (error) {
            console.error("Error initializing Firebase:", error);
            showMessage("خطأ في تهيئة Firebase. يرجى التحقق من الإعدادات.", "error");
            setLoading(false);
        }
    }, []);

    const contextValue = { db, auth, userId, loading, showMessage };

    return (
        <AppContext.Provider value={contextValue}>
            {children}
            <MessageModal message={message} type={messageType} onClose={() => setMessage('')} />
        </AppContext.Provider>
    );
};

// Component to display and manage a list of items (Correspondence, Activities, Reports, HR)
const ItemList = ({ governorateId, collectionName, fields, formComponent: FormComponent, emptyMessage }) => {
    const { db, showMessage, userId } = useAppContext();
    const [items, setItems] = useState([]);
    const [showForm, setShowForm] = useState(false);
    const [searchTerm, setSearchTerm] = useState('');
    const [searchDate, setSearchDate] = useState('');
    const [searchNumber, setSearchNumber] = useState(''); // New state for number search
    const [filteredItems, setFilteredItems] = useState([]);
    const [showUpdateStatusModal, setShowUpdateStatusModal] = useState(false);
    const [selectedCorrespondence, setSelectedCorrespondence] = useState(null);

    useEffect(() => {
        if (!db || !governorateId) return;

        // Construct the collection path based on whether it's public or user-specific
        const path = `artifacts/${__app_id}/public/data/${collectionName}`;
        console.log(`Firestore: Attempting to fetch from path: ${path}`); // Log the path
        const q = query(collection(db, path), orderBy('timestamp', 'desc'));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const fetchedItems = snapshot.docs
                .map(doc => ({ id: doc.id, ...doc.data() }))
                .filter(item => item.governorateId === governorateId); // Filter by governorateId
            setItems(fetchedItems);
        }, (error) => {
            console.error("Error fetching items:", error);
            showMessage(`خطأ في جلب ${emptyMessage.toLowerCase()}: ${error.message}`, "error");
        });

        return () => unsubscribe();
    }, [db, governorateId, collectionName, showMessage]);

    // Effect to filter items whenever items, searchTerm, searchDate, or searchNumber changes
    useEffect(() => {
        let currentFilteredItems = items;

        if (searchTerm) {
            const lowerCaseSearchTerm = searchTerm.toLowerCase();
            currentFilteredItems = currentFilteredItems.filter(item =>
                fields.some(field =>
                    item[field.key] && String(item[field.key]).toLowerCase().includes(lowerCaseSearchTerm)
                )
            );
        }

        if (searchDate) {
            currentFilteredItems = currentFilteredItems.filter(item => {
                if (item.date && item.date.toDate) { // Check if it's a Firestore Timestamp
                    return item.date.toDate().toISOString().split('T')[0] === searchDate;
                } else if (item.date) { // Assume it's a string date
                    return item.date === searchDate;
                }
                return false;
            });
        }

        // Apply number search only for 'correspondence' and 'circulars'
        if (searchNumber && (collectionName === 'correspondence' || collectionName === 'circulars' || collectionName === 'legalCases')) {
            const lowerCaseSearchNumber = searchNumber.toLowerCase();
            currentFilteredItems = currentFilteredItems.filter(item => {
                if (collectionName === 'correspondence' && item.correspondenceNumber) {
                    return String(item.correspondenceNumber).toLowerCase().includes(lowerCaseSearchNumber);
                }
                if (collectionName === 'circulars' && item.circularNumber) {
                    return String(item.circularNumber).toLowerCase().includes(lowerCaseSearchNumber);
                }
                if (collectionName === 'legalCases' && item.caseNumber) {
                    return String(item.caseNumber).toLowerCase().includes(lowerCaseSearchNumber);
                }
                return false;
            });
        }

        setFilteredItems(currentFilteredItems);
    }, [items, searchTerm, searchDate, searchNumber, fields, collectionName]);


    // Handle adding a new item
    const handleAddItem = async (newItem) => {
        try {
            const path = `artifacts/${__app_id}/public/data/${collectionName}`;
            console.log(`Firestore: Attempting to add to path: ${path}`); // Log the path
            await addDoc(collection(db, path), {
                ...newItem,
                governorateId,
                userId, // Store the user ID who added the item
                timestamp: serverTimestamp()
            });
            showMessage(`${emptyMessage} أضيفت بنجاح!`);
            setShowForm(false);
        } catch (e) {
            console.error("Error adding document: ", e);
            showMessage(`خطأ في إضافة ${emptyMessage.toLowerCase()}: ${e.message}`, "error");
        }
    };

    // Handle deleting an item
    const handleDeleteItem = async (id) => {
        if (window.confirm("هل أنت متأكد أنك تريد حذف هذا العنصر؟")) {
            try {
                const path = `artifacts/${__app_id}/public/data/${collectionName}`;
                console.log(`Firestore: Attempting to delete from path: ${path}/${id}`); // Log the path
                await deleteDoc(doc(db, path, id));
                showMessage(`${emptyMessage} تم حذفها بنجاح.`);
            } catch (e) {
                console.error("Error deleting document: ", e);
                showMessage(`خطأ في حذف ${emptyMessage.toLowerCase()}: ${e.message}`, "error");
            }
        }
    };

    // Handle updating correspondence status
    const handleUpdateCorrespondenceStatus = async (correspondenceId, newStatus, comment) => {
        try {
            const path = `artifacts/${__app_id}/public/data/correspondence`;
            const docRef = doc(db, path, correspondenceId);
            await updateDoc(docRef, {
                status: newStatus,
                history: arrayUnion({
                    action: 'تغيير الحالة',
                    comment: comment,
                    timestamp: serverTimestamp(),
                    changedBy: userId
                })
            });
            showMessage(`تم تحديث حالة المراسلة بنجاح إلى: ${newStatus}`);
            setShowUpdateStatusModal(false);
            setSelectedCorrespondence(null);
        } catch (e) {
            console.error("Error updating correspondence status: ", e);
            showMessage(`خطأ في تحديث حالة المراسلة: ${e.message}`, "error");
        }
    };

    const handleClearSearch = () => {
        setSearchTerm('');
        setSearchDate('');
        setSearchNumber(''); // Clear number search as well
    };

    return (
        <div className="p-6 bg-white rounded-lg shadow-md">
            <h3 className="text-2xl font-bold text-gray-800 mb-6 border-b pb-3">
                {collectionName === 'correspondence' ? 'إدارة المراسلات' :
                 collectionName === 'activities' ? 'إدارة الأنشطة' :
                 collectionName === 'reports' ? 'إدارة التقارير' :
                 collectionName === 'circulars' ? 'إدارة التعميمات' :
                 collectionName === 'meetings' ? 'إدارة الاجتماعات' :
                 collectionName === 'visits' ? 'إدارة الزيارات' :
                 collectionName === 'humanResources' ? 'إدارة الموارد البشرية' :
                 collectionName === 'accounts' ? 'إدارة الحسابات' :
                 collectionName === 'trainingCourses' ? 'إدارة الدورات التدريبية' :
                 collectionName === 'servicesMechanisms' ? 'إدارة الخدمات والآليات' :
                 collectionName === 'religiousEducationSchools' ? 'إدارة المدارس الدينية' :
                 collectionName === 'generalEducationSchools' ? 'إدارة المدارس العامة' :
                 collectionName === 'religiousCharitableInstitutions' ? 'إدارة المؤسسات الدينية والخيرية' :
                 collectionName === 'endowedInvestments' ? 'إدارة الأملاك الموقوفة والاستثمار' :
                 collectionName === 'engineeringProjects' ? 'إدارة المشاريع الهندسية' :
                 collectionName === 'legalCases' ? 'إدارة القضايا القانونية' :
                 'إدارة البيانات'}
            </h3>

            {/* Search Section */}
            <div className="mb-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
                <h4 className="text-lg font-semibold text-gray-700 mb-4">بحث</h4>
                <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-4">
                    <div>
                        <label htmlFor={`${collectionName}-searchTerm`} className="block text-gray-700 text-sm font-bold mb-2">
                            بحث نصي (الموضوع، الوصف، العنوان، إلخ):
                        </label>
                        <input
                            type="text"
                            id={`${collectionName}-searchTerm`}
                            value={searchTerm}
                            onChange={(e) => setSearchTerm(e.target.value)}
                            placeholder="ابحث عن طريق الكلمات المفتاحية..."
                            className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                            dir="rtl"
                        />
                    </div>
                    <div>
                        <label htmlFor={`${collectionName}-searchDate`} className="block text-gray-700 text-sm font-bold mb-2">
                            بحث بالتاريخ:
                        </label>
                        <input
                            type="date"
                            id={`${collectionName}-searchDate`}
                            value={searchDate}
                            onChange={(e) => setSearchDate(e.target.value)}
                            className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                        />
                    </div>
                    {/* Number Search Field - Conditional for specific collections */}
                    {(collectionName === 'correspondence' || collectionName === 'circulars' || collectionName === 'legalCases') && (
                        <div>
                            <label htmlFor={`${collectionName}-searchNumber`} className="block text-gray-700 text-sm font-bold mb-2">
                                بحث بالرقم ({collectionName === 'correspondence' ? 'رقم الكتاب' : collectionName === 'circulars' ? 'رقم التعميم' : 'رقم القضية'}):
                            </label>
                            <input
                                type="text"
                                id={`${collectionName}-searchNumber`}
                                value={searchNumber}
                                onChange={(e) => setSearchNumber(e.target.value)}
                                placeholder={`ابحث عن طريق رقم ${collectionName === 'correspondence' ? 'الكتاب' : collectionName === 'circulars' ? 'التعميم' : 'القضية'}...`}
                                className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                                dir="rtl"
                            />
                        </div>
                    )}
                </div>
                <div className="flex justify-end space-x-2 space-x-reverse">
                    <button
                        onClick={handleClearSearch}
                        className="px-4 py-2 bg-gray-400 text-white rounded-lg hover:bg-gray-500 transition duration-200 focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-opacity-50"
                    >
                        مسح البحث
                    </button>
                </div>
            </div>

            {/* Instruction for adding items */}
            <div className="bg-blue-50 border-l-4 border-blue-400 text-blue-700 p-4 mb-6 rounded-md" role="alert" dir="rtl">
                <p className="font-bold">ملاحظة:</p>
                <p>لإضافة {emptyMessage} جديدة، انقر على زر "إضافة {emptyMessage} جديدة" أدناه. سيظهر نموذج الإدخال.</p>
            </div>

            <button
                onClick={() => setShowForm(!showForm)}
                className="mb-6 px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition duration-200 shadow-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-50"
            >
                {showForm ? `إلغاء إضافة ${emptyMessage}` : `إضافة ${emptyMessage} جديدة`}
            </button>

            {showForm && (
                <div className="mb-8 p-6 bg-gray-50 rounded-lg border border-gray-200">
                    <FormComponent onSubmit={handleAddItem} onCancel={() => setShowForm(false)} />
                </div>
            )}

            {filteredItems.length === 0 && (searchTerm || searchDate || searchNumber) ? (
                <p className="text-gray-600 text-center py-8">لا توجد نتائج مطابقة لمعايير البحث.</p>
            ) : filteredItems.length === 0 ? (
                <p className="text-gray-600 text-center py-8">{`لا توجد ${emptyMessage.toLowerCase()} متاحة لهذه المحافظة.`}</p>
            ) : (
                <div className="space-y-4">
                    {filteredItems.map(item => (
                        <div key={item.id} className="bg-gray-100 p-5 rounded-lg shadow-sm border border-gray-200 flex flex-col md:flex-row justify-between items-start md:items-center">
                            <div className="flex-grow mb-4 md:mb-0">
                                {fields.map(field => (
                                    <p key={field.key} className="text-gray-700 text-right mb-1">
                                        <span className="font-semibold text-gray-800">{field.label}: </span>
                                        {/* Render date fields correctly */}
                                        {field.key === 'date' || field.key === 'hireDate' || field.key === 'startDate' || field.key === 'endDate' || field.key === 'expectedEndDate' || field.key === 'filingDate' || field.key === 'resolutionDate' ? (item[field.key]?.toDate ? item[field.key].toDate().toLocaleDateString('ar-EG') : item[field.key]) : item[field.key]}
                                        {field.key === 'time' && item[field.key]}
                                        {/* Render certificates as a list if available */}
                                        {field.key === 'certificates' && item.certificates && item.certificates.length > 0 && (
                                            <ul className="list-disc list-inside text-sm text-gray-600 mt-1">
                                                {item.certificates.map((cert, certIndex) => (
                                                    <li key={certIndex} className="mb-0.5">
                                                        {cert.certificateName} ({cert.issueDate}) من {cert.issuingBody}
                                                    </li>
                                                ))}
                                            </ul>
                                        )}
                                        {/* Render lastUpdated for accounts */}
                                        {field.key === 'lastUpdated' && item.lastUpdated && (
                                            <span>
                                                {item.lastUpdated?.toDate ? item.lastUpdated.toDate().toLocaleString('ar-EG') : 'N/A'}
                                            </span>
                                        )}
                                    </p>
                                ))}
                                {/* Display current status if it's a correspondence */}
                                {collectionName === 'correspondence' && item.status && (
                                    <p className="text-gray-700 text-right mb-1">
                                        <span className="font-semibold text-gray-800">الحالة الحالية: </span>
                                        <span className={`px-2 py-1 rounded-full text-sm font-medium ${
                                            item.status === 'قيد المراجعة' ? 'bg-yellow-200 text-yellow-800' :
                                            item.status === 'تمت الموافقة' ? 'bg-green-200 text-green-800' :
                                            item.status === 'مرفوضة' ? 'bg-red-200 text-red-800' :
                                            'bg-gray-200 text-gray-800'
                                        }`}>
                                            {item.status}
                                        </span>
                                    </p>
                                )}
                                {/* Display History if it's a correspondence and has history */}
                                {collectionName === 'correspondence' && item.history && item.history.length > 0 && (
                                    <div className="mt-4 pt-4 border-t border-gray-300">
                                        <p className="font-semibold text-gray-800 mb-2">سجل التغييرات:</p>
                                        <ul className="list-disc list-inside text-sm text-gray-600">
                                            {item.history.map((entry, index) => (
                                                <li key={index} className="mb-1 text-right">
                                                    <span className="font-medium">{entry.action}: </span>
                                                    {entry.comment}
                                                    <span className="text-xs text-gray-500 mr-2">
                                                        ({entry.timestamp?.toDate ? entry.timestamp.toDate().toLocaleString('ar-EG') : 'N/A'} بواسطة {entry.changedBy || 'مجهول'})
                                                    </span>
                                                </li>
                                            ))}
                                        </ul>
                                    </div>
                                )}
                                <p className="text-sm text-gray-500 text-right mt-2">
                                    أضيف بواسطة: {item.userId || 'مجهول'}
                                </p>
                            </div>
                            <div className="flex flex-col md:flex-row space-y-2 md:space-y-0 md:space-x-2 md:space-x-reverse mt-4 md:mt-0">
                                {collectionName === 'correspondence' && (
                                    <button
                                        onClick={() => {
                                            setSelectedCorrespondence(item);
                                            setShowUpdateStatusModal(true);
                                        }}
                                        className="px-4 py-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700 transition duration-200 focus:outline-none focus:ring-2 focus:ring-purple-500 focus:ring-opacity-50"
                                    >
                                        تحديث الحالة
                                    </button>
                                )}
                                <button
                                    onClick={() => handleDeleteItem(item.id)}
                                    className="px-4 py-2 bg-red-500 text-white rounded-lg hover:bg-red-600 transition duration-200 focus:outline-none focus:ring-2 focus:ring-red-400 focus:ring-opacity-50"
                                >
                                    حذف
                                </button>
                            </div>
                        </div>
                    ))}
                </div>
            )}

            {showUpdateStatusModal && selectedCorrespondence && (
                <UpdateCorrespondenceStatusModal
                    correspondence={selectedCorrespondence}
                    onUpdate={handleUpdateCorrespondenceStatus}
                    onClose={() => {
                        setShowUpdateStatusModal(false);
                        setSelectedCorrespondence(null);
                    }}
                />
            )}
        </div>
    );
};

// Form for adding Correspondence
const CorrespondenceForm = ({ onSubmit, onCancel }) => {
    const [type, setType] = useState('وارد');
    const [subject, setSubject] = useState('');
    const [correspondenceNumber, setCorrespondenceNumber] = useState(''); // New state for correspondence number
    const [date, setDate] = useState('');
    const [details, setDetails] = useState('');
    const [status, setStatus] = useState('قيد المراجعة'); // Initial status for new correspondence

    const handleSubmit = (e) => {
        e.preventDefault();
        if (!subject || !correspondenceNumber || !date || !details || !status) {
            alert('الرجاء ملء جميع الحقول.'); // Using alert for simplicity, but a custom modal is preferred in production
            return;
        }
        // Initial history entry
        const initialHistory = [{
            action: 'إنشاء الكتاب',
            comment: `تم إنشاء الكتاب بحالة: ${status}`,
            timestamp: serverTimestamp(),
            changedBy: useAppContext().userId // Get current user ID from context
        }];

        onSubmit({ type, subject, correspondenceNumber, date, details, status, history: initialHistory });
        // Clear form
        setType('وارد');
        setSubject('');
        setCorrespondenceNumber('');
        setDate('');
        setDetails('');
        setStatus('قيد المراجعة');
    };

    return (
        <form onSubmit={handleSubmit} className="space-y-4 text-right">
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="type">
                    النوع:
                </label>
                <select
                    id="type"
                    value={type}
                    onChange={(e) => setType(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                >
                    <option value="وارد">وارد</option>
                    <option value="صادر">صادر</option>
                </select>
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="subject">
                    الموضوع:
                </label>
                <input
                    type="text"
                    id="subject"
                    value={subject}
                    onChange={(e) => setSubject(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                    dir="rtl"
                />
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="correspondenceNumber">
                    رقم الكتاب:
                </label>
                <input
                    type="text"
                    id="correspondenceNumber"
                    value={correspondenceNumber}
                    onChange={(e) => setCorrespondenceNumber(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                    dir="rtl"
                />
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="date">
                    التاريخ:
                </label>
                <input
                    type="date"
                    id="date"
                    value={date}
                    onChange={(e) => setDate(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                />
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="details">
                    التفاصيل:
                </label>
                <textarea
                    id="details"
                    value={details}
                    onChange={(e) => setDetails(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline h-24"
                    dir="rtl"
                ></textarea>
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="status">
                    الحالة الأولية:
                </label>
                <select
                    id="status"
                    value={status}
                    onChange={(e) => setStatus(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                >
                    <option value="قيد المراجعة">قيد المراجعة</option>
                    <option value="تمت الموافقة">تمت الموافقة</option>
                    <option value="تمت الإحالة">تمت الإحالة</option>
                    <option value="مرفوضة">مرفوضة</option>
                    <option value="مكتملة">مكتملة</option>
                </select>
            </div>
            <div className="flex justify-end space-x-2 space-x-reverse">
                <button
                    type="submit"
                    className="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline transition duration-200"
                >
                    إضافة مراسلة
                </button>
                <button
                    type="button"
                    onClick={onCancel}
                    className="bg-gray-400 hover:bg-gray-500 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline transition duration-200"
                >
                    إلغاء
                </button>
            </div>
        </form>
    );
};

// Form for updating Correspondence Status
const UpdateCorrespondenceStatusModal = ({ correspondence, onUpdate, onClose }) => {
    const [newStatus, setNewStatus] = useState(correspondence.status || 'قيد المراجعة');
    const [comment, setComment] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (!newStatus) {
            alert('الرجاء اختيار حالة جديدة.');
            return;
        }
        onUpdate(correspondence.id, newStatus, comment);
    };

    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="relative bg-white rounded-lg shadow-xl p-6 max-w-lg w-full text-right">
                <h3 className="text-2xl font-bold text-gray-800 mb-6 border-b pb-3">تحديث حالة المراسلة</h3>
                <p className="text-gray-700 mb-4">
                    <span className="font-semibold">رقم الكتاب:</span> {correspondence.correspondenceNumber}
                </p>
                <p className="text-gray-700 mb-6">
                    <span className="font-semibold">الموضوع:</span> {correspondence.subject}
                </p>

                <form onSubmit={handleSubmit} className="space-y-4">
                    <div>
                        <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="newStatus">
                            الحالة الجديدة:
                        </label>
                        <select
                            id="newStatus"
                            value={newStatus}
                            onChange={(e) => setNewStatus(e.target.value)}
                            className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                        >
                            <option value="قيد المراجعة">قيد المراجعة</option>
                            <option value="تمت الموافقة">تمت الموافقة</option>
                            <option value="تمت الإحالة">تمت الإحالة</option>
                            <option value="مرفوضة">مرفوضة</option>
                    <option value="مكتملة">مكتملة</option>
                </select>
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="comment">
                    تعليق (اختياري):
                </label>
                <textarea
                    id="comment"
                    value={comment}
                    onChange={(e) => setComment(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline h-20"
                    dir="rtl"
                ></textarea>
            </div>
            <div className="flex justify-end space-x-2 space-x-reverse mt-6">
                <button
                    type="submit"
                    className="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline transition duration-200"
                >
                    تحديث
                </button>
                <button
                    type="button"
                    onClick={onClose}
                    className="bg-gray-400 hover:bg-gray-500 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline transition duration-200"
                >
                    إلغاء
                </button>
            </div>
        </form>
    </div>
</div>
    );
};


// Form for adding Activity
const ActivityForm = ({ onSubmit, onCancel }) => {
    const [name, setName] = useState('');
    const [date, setDate] = useState('');
    const [description, setDescription] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (!name || !date || !description) {
            alert('الرجاء ملء جميع الحقول.');
            return;
        }
        onSubmit({ name, date, description });
        setName('');
        setDate('');
        setDescription('');
    };

    return (
        <form onSubmit={handleSubmit} className="space-y-4 text-right">
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="activityName">
                    اسم النشاط:
                </label>
                <input
                    type="text"
                    id="activityName"
                    value={name}
                    onChange={(e) => setName(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                    dir="rtl"
                />
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="activityDate">
                    التاريخ:
                </label>
                <input
                    type="date"
                    id="activityDate"
                    value={date}
                    onChange={(e) => setDate(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                />
            </div>
            <div>
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="activityDescription">
                    الوصف:
                </label>
                <textarea
                    id="activityDescription"
                    value={description}
                    onChange={(e) => setDescription(e.target.value)}
                    className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline h-24"
                    dir="rtl"
                ></textarea>
            </div>
            <div className="flex justify-end space-x-2 space-x-reverse">
                <button
                    type="submit"
                    className="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline transition duration-200"
                >
                    إضافة نشاط
                </button>
                <button
                    type="button"
                    onClick={onCancel}
                    className="bg-gray-400 hover:bg-gray-500 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline transition duration-200"
                >
                    إلغاء
                </button>
            </div>
        </form>
    );
};

// Form for adding Report
const ReportForm = ({ onSubmit, onCancel }) => {
    const [title, setTitle] = useState('');
    const [date, setDate] = useState('');
    const [summary, setSummary] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (!title || !date || !summary) {
            alert('الرجاء ملء جميع الحقول.');
            return;
        }
        onSubmit({ title, date, summary });
        setTitle('');
        setD# aawqaf
