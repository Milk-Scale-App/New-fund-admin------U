// Firebase Configuration - YOUR CONFIG
const firebaseConfig = {
    apiKey: "AIzaSyD8w1L_Nxe5UiPhpAe1rbyDo4KYb-pH0VU",
    authDomain: "new-fund-money.firebaseapp.com",
    projectId: "new-fund-money",
    storageBucket: "new-fund-money.firebasestorage.app",
    messagingSenderId: "912524669808",
    appId: "1:912524669808:web:48f925b308a95832d495b5",
    measurementId: "G-4JDE7VM0MC"
};

// Initialize Firebase
firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const db = firebase.firestore();

// Admin Configuration
const ADMIN_PASSWORD = "Winner@#2008";
const GAME_CYCLE_DURATION = 30;

// Global Variables
let selectedUsers = new Set();
let gameControlListener = null;
let usersListener = null;

// Collections
const USERS_COLLECTION = 'users';
const GAME_CONTROL_COLLECTION = 'gameControl';
const BETS_COLLECTION = 'bets';
const ADD_MONEY_COLLECTION = 'addMoneyRequests';
const WITHDRAWAL_COLLECTION = 'withdrawalRequests';
const NOTIFICATIONS_COLLECTION = 'notifications';

// User Status Constants
const USER_STATUS = {
    ACTIVE: 'active',
    BLOCKED: 'blocked',
    DELETED: 'deleted'
};

// Password Protection
function checkAdminPassword() {
    const enteredPassword = document.getElementById('adminPassword').value;
    const errorElement = document.getElementById('passwordError');
    
    if (enteredPassword === ADMIN_PASSWORD) {
        document.getElementById('passwordOverlay').classList.remove('active');
        document.getElementById('adminContent').classList.add('active');
        initializeAdminPanel();
    } else {
        errorElement.style.display = 'block';
        document.getElementById('adminPassword').value = '';
        document.getElementById('adminPassword').focus();
    }
}

// Admin Panel Initialization
function initializeAdminPanel() {
    setupGameControlListener();
    loadAllUsers();
    loadAddMoneyRequests();
    loadWithdrawalRequests();
    loadAnalytics();
    loadNotificationsHistory();
    
    // Update admin status
    updateAdminStatus('Connected to Firebase');
}

function updateAdminStatus(message) {
    document.getElementById('adminStatus').textContent = message;
}

// Section Navigation
function showAdminSection(sectionId) {
    // Hide all sections
    document.querySelectorAll('.admin-section').forEach(section => {
        section.classList.remove('active');
    });
    
    // Remove active class from all sidebar buttons
    document.querySelectorAll('.sidebar-btn').forEach(btn => {
        btn.classList.remove('active');
    });
    
    // Show selected section
    document.getElementById(sectionId).classList.add('active');
    
    // Activate corresponding sidebar button
    document.querySelector(`.sidebar-btn[onclick="showAdminSection('${sectionId}')"]`).classList.add('active');
}

// Game Control Functions
function setupGameControlListener() {
    if (gameControlListener) {
        gameControlListener();
    }
    
    gameControlListener = db.collection(GAME_CONTROL_COLLECTION).doc('current')
        .onSnapshot((doc) => {
            if (doc.exists) {
                const gameData = doc.data();
                updateGameControlDisplay(gameData);
            } else {
                // Initialize game control document if it doesn't exist
                initializeGameControl();
            }
        }, (error) => {
            console.error('Game control listener error:', error);
            updateAdminStatus('Connection Error');
        });
}

function initializeGameControl() {
    const initialGameData = {
        cycleId: generateCycleId(),
        lastResult: 'none',
        nextResult: 'none',
        timerEnd: firebase.firestore.Timestamp.fromMillis(Date.now() + (GAME_CYCLE_DURATION * 1000)),
        isProcessing: false,
        totalBets: 0,
        totalPayout: 0,
        createdAt: firebase.firestore.FieldValue.serverTimestamp()
    };
    
    db.collection(GAME_CONTROL_COLLECTION).doc('current').set(initialGameData)
        .then(() => {
            console.log('Game control initialized');
        })
        .catch((error) => {
            console.error('Game control initialization error:', error);
        });
}

function updateGameControlDisplay(gameData) {
    // Update cycle ID
    document.getElementById('current-cycle-id').textContent = gameData.cycleId || '-';
    
    // Update timer
    if (gameData.timerEnd) {
        const now = Date.now();
        const timerEnd = gameData.timerEnd.toMillis();
        const timeLeft = Math.max(0, (timerEnd - now) / 1000);
        document.getElementById('current-timer').textContent = timeLeft.toFixed(1) + 's';
    }
    
    // Update results
    document.getElementById('last-result-display').textContent = 
        gameData.lastResult === 'none' ? '-' : gameData.lastResult;
    document.getElementById('next-result-display').textContent = 
        gameData.nextResult === 'none' ? '-' : gameData.nextResult;
    
    // Update stats
    document.getElementById('total-bets').textContent = gameData.totalBets || 0;
    document.getElementById('total-payout').textContent = gameData.totalPayout || 0;
    
    // Update active players count (this would need to be calculated)
    updateActivePlayersCount();
}

function generateCycleId() {
    return 'CYCLE_' + Date.now().toString(36).toUpperCase();
}

async function setNextResult(result) {
    try {
        if (!confirm(`Set next result to ${result.toUpperCase()}? This will process current round and start new game.`)) {
            return;
        }
        
        showAdminLoading('Setting result...');
        
        // Get current game control data
        const controlDoc = await db.collection(GAME_CONTROL_COLLECTION).doc('current').get();
        const currentData = controlDoc.data();
        
        if (currentData.isProcessing) {
            throw new Error('Game is currently processing. Please wait.');
        }
        
        // Set processing flag
        await db.collection(GAME_CONTROL_COLLECTION).doc('current').update({
            isProcessing: true
        });
        
        // Process auto payout for previous round if there was a result
        if (currentData.lastResult && currentData.lastResult !== 'none') {
            await processAutoPayout(currentData.lastResult, currentData.cycleId);
        }
        
        // Start new game cycle
        const newCycleId = generateCycleId();
        const timerEnd = firebase.firestore.Timestamp.fromMillis(
            Date.now() + (GAME_CYCLE_DURATION * 1000)
        );
        
        const updateData = {
            lastResult: currentData.nextResult !== 'none' ? currentData.nextResult : 'none',
            nextResult: result,
            cycleId: newCycleId,
            timerEnd: timerEnd,
            isProcessing: false,
            updatedAt: firebase.firestore.FieldValue.serverTimestamp()
        };
        
        await db.collection(GAME_CONTROL_COLLECTION).doc('current').set(updateData, { merge: true });
        
        hideAdminLoading();
        showAdminMessage(`Next result set to '${result}'`, true);
        
        // Reload game stats
        loadGameStats();
        
    } catch (error) {
        hideAdminLoading();
        console.error('Set result error:', error);
        showAdminMessage('Failed to set result: ' + error.message, false);
        
        // Reset processing flag on error
        await db.collection(GAME_CONTROL_COLLECTION).doc('current').update({
            isProcessing: false
        });
    }
}

async function processAutoPayout(winningColor, cycleId) {
    try {
        // Get all pending bets for this cycle
        const betsSnapshot = await db.collection(BETS_COLLECTION)
            .where('cycleId', '==', cycleId)
            .where('status', '==', 'pending')
            .get();
        
        if (betsSnapshot.empty) {
            console.log('No pending bets to process');
            return { winners: 0, totalPayout: 0 };
        }
        
        const batch = db.batch();
        const userUpdates = {};
        let winnersCount = 0;
        let totalPayout = 0;
        
        // Process each bet
        betsSnapshot.docs.forEach(doc => {
            const bet = doc.data();
            const betRef = doc.ref;
            
            if (bet.color === winningColor) {
                // Winner - 2x payout
                const winAmount = bet.amount * 2;
                winnersCount++;
                totalPayout += winAmount;
                
                // Track user balance changes
                if (!userUpdates[bet.userId]) {
                    userUpdates[bet.userId] = {
                        balanceChange: 0,
                        bets: []
                    };
                }
                userUpdates[bet.userId].balanceChange += winAmount;
                userUpdates[bet.userId].bets.push(betRef);
                
                // Update bet status
                batch.update(betRef, {
                    status: 'won',
                    winAmount: winAmount,
                    processedAt: firebase.firestore.FieldValue.serverTimestamp()
                });
                
            } else {
                // Loser - mark as lost
                batch.update(betRef, {
                    status: 'lost',
                    processedAt: firebase.firestore.FieldValue.serverTimestamp()
                });
            }
        });
        
        // Update user balances
        for (const [userId, data] of Object.entries(userUpdates)) {
            const userRef = db.collection(USERS_COLLECTION).doc(userId);
            batch.update(userRef, {
                balance: firebase.firestore.FieldValue.increment(data.balanceChange),
                lastUpdated: firebase.firestore.FieldValue.serverTimestamp()
            });
        }
        
        // Commit all changes
        await batch.commit();
        
        // Update game control with payout info
        await db.collection(GAME_CONTROL_COLLECTION).doc('current').update({
            totalPayout: firebase.firestore.FieldValue.increment(totalPayout),
            lastPayout: totalPayout,
            lastWinners: winnersCount
        });
        
        console.log(`Auto payout processed: ${winnersCount} winners, ₹${totalPayout} payout`);
        return { winners: winnersCount, totalPayout: totalPayout };
        
    } catch (error) {
        console.error('Auto payout error:', error);
        throw error;
    }
}

async function loadGameStats() {
    try {
        // Calculate total bets (you might want to aggregate this differently)
        const today = new Date();
        today.setHours(0, 0, 0, 0);
        
        const betsSnapshot = await db.collection(BETS_COLLECTION)
            .where('placedAt', '>=', firebase.firestore.Timestamp.fromDate(today))
            .get();
        
        const totalBets = betsSnapshot.size;
        document.getElementById('total-bets').textContent = totalBets;
        
    } catch (error) {
        console.error('Load game stats error:', error);
    }
}

async function updateActivePlayersCount() {
    try {
        // Count users who have logged in today
        const today = new Date();
        today.setHours(0, 0, 0, 0);
        
        const usersSnapshot = await db.collection(USERS_COLLECTION)
            .where('lastLogin', '>=', firebase.firestore.Timestamp.fromDate(today))
            .where('status', '==', 'active')
            .get();
        
        document.getElementById('active-players').textContent = usersSnapshot.size;
        
    } catch (error) {
        console.error('Update active players error:', error);
    }
}

// User Management Functions
function loadAllUsers() {
    if (usersListener) {
        usersListener();
    }
    
    usersListener = db.collection(USERS_COLLECTION)
        .orderBy('createdAt', 'desc')
        .onSnapshot((snapshot) => {
            const usersList = document.getElementById('users-list');
            usersList.innerHTML = '';
            
            let totalUsers = 0;
            let activeUsers = 0;
            let blockedUsers = 0;
            let deletedUsers = 0;
            
            snapshot.forEach(doc => {
                const user = doc.data();
                user.id = doc.id;
                renderUserItem(user);
                
                totalUsers++;
                if (user.status === 'active') activeUsers++;
                if (user.status === 'blocked') blockedUsers++;
                if (user.status === 'deleted') deletedUsers++;
            });
            
            // Update statistics
            document.getElementById('total-users').textContent = totalUsers;
            document.getElementById('active-users').textContent = activeUsers;
            document.getElementById('blocked-users').textContent = blockedUsers;
            document.getElementById('deleted-users').textContent = deletedUsers;
            
        }, (error) => {
            console.error('Users listener error:', error);
            document.getElementById('users-list').innerHTML = 
                '<div class="loading-message">Error loading users</div>';
        });
}

function renderUserItem(user) {
    const usersList = document.getElementById('users-list');
    
    const userItem = document.createElement('div');
    userItem.className = 'user-item';
    userItem.innerHTML = `
        <div class="user-info">
            <input type="checkbox" class="user-checkbox" 
                   onchange="toggleUserSelection('${user.id}')"
                   ${user.status === 'deleted' ? 'disabled' : ''}>
            <div class="user-main-info">
                <div class="user-name">${user.name || 'Unknown User'}</div>
                <div class="user-email">${user.email || 'No email'}</div>
                <div class="user-id">ID: ${user.userId || 'N/A'}</div>
            </div>
        </div>
        <div class="user-stats">
            <div class="user-balance">₹${user.balance || 0}</div>
            <div class="user-status status-${user.status || 'active'}">
                ${user.status || 'active'}
            </div>
        </div>
        <div class="user-actions">
            <button class="action-btn view-btn" onclick="viewUserDetails('${user.id}')">
                View
            </button>
            <button class="action-btn edit-btn" onclick="editUser('${user.id}')">
                Edit
            </button>
            <button class="action-btn delete-btn" onclick="deleteUser('${user.id}')">
                Delete
            </button>
        </div>
    `;
    
    usersList.appendChild(userItem);
}

function toggleUserSelection(userId) {
    if (selectedUsers.has(userId)) {
        selectedUsers.delete(userId);
    } else {
        selectedUsers.add(userId);
    }
    
    document.getElementById('selected-count').textContent = selectedUsers.size;
}

function clearSelection() {
    selectedUsers.clear();
    document.getElementById('selected-count').textContent = 0;
    
    // Uncheck all checkboxes
    document.querySelectorAll('.user-checkbox').forEach(checkbox => {
        checkbox.checked = false;
    });
}

async function viewUserDetails(userId) {
    try {
        const userDoc = await db.collection(USERS_COLLECTION).doc(userId).get();
        if (!userDoc.exists) {
            showAdminMessage('User not found', false);
            return;
        }
        
        const user = userDoc.data();
        const modalContent = document.getElementById('user-details-content');
        
        modalContent.innerHTML = `
            <div class="user-detail-section">
                <h4>Basic Information</h4>
                <p><strong>Name:</strong> ${user.name || 'N/A'}</p>
                <p><strong>Email:</strong> ${user.email || 'N/A'}</p>
                <p><strong>User ID:</strong> ${user.userId || 'N/A'}</p>
                <p><strong>Status:</strong> <span class="user-status status-${user.status}">${user.status}</span></p>
                <p><strong>Balance:</strong> ₹${user.balance || 0}</p>
                <p><strong>Joined:</strong> ${user.createdAt ? user.createdAt.toDate().toLocaleString() : 'N/A'}</p>
                <p><strong>Last Login:</strong> ${user.lastLogin ? user.lastLogin.toDate().toLocaleString() : 'N/A'}</p>
            </div>
            
            ${user.bankDetails ? `
            <div class="user-detail-section">
                <h4>Bank Details</h4>
                <p><strong>Account Holder:</strong> ${user.bankDetails.accountHolder}</p>
                <p><strong>Account Number:</strong> ${user.bankDetails.accountNumber}</p>
                <p><strong>IFSC Code:</strong> ${user.bankDetails.ifscCode}</p>
                <p><strong>Bank Name:</strong> ${user.bankDetails.bankName}</p>
            </div>
            ` : ''}
            
            <div class="user-detail-section">
                <h4>Quick Actions</h4>
                <div class="action-buttons">
                    <button class="bulk-btn ${user.status === 'active' ? 'block-btn' : 'unblock-btn'}" 
                            onclick="${user.status === 'active' ? `blockUser('${userId}')` : `unblockUser('${userId}')`}">
                        ${user.status === 'active' ? 'Block User' : 'Unblock User'}
                    </button>
                    <button class="bulk-btn delete-btn" onclick="deleteUser('${userId}')">
                        Delete User
                    </button>
                    <button class="bulk-btn reset-btn" onclick="resetUserBalance('${userId}')">
                        Reset Balance
                    </button>
                </div>
            </div>
            
            <div class="user-detail-section">
                <h4>Balance Management</h4>
                <div class="balance-control">
                    <input type="number" id="new-balance-${userId}" placeholder="New balance" value="${user.balance || 0}">
                    <button onclick="updateUserBalance('${userId}')">Update Balance</button>
                </div>
            </div>
        `;
        
        document.getElementById('user-details-modal').classList.remove('hidden');
        
    } catch (error) {
        console.error('View user details error:', error);
        showAdminMessage('Error loading user details', false);
    }
}

function closeUserModal() {
    document.getElementById('user-details-modal').classList.add('hidden');
}

async function updateUserBalance(userId) {
    const newBalance = parseInt(document.getElementById(`new-balance-${userId}`).value);
    
    if (isNaN(newBalance) || newBalance < 0) {
        showAdminMessage('Please enter a valid balance', false);
        return;
    }
    
    try {
        await db.collection(USERS_COLLECTION).doc(userId).update({
            balance: newBalance,
            lastUpdated: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        showAdminMessage('User balance updated successfully', true);
        closeUserModal();
        
    } catch (error) {
        console.error('Update balance error:', error);
        showAdminMessage('Error updating balance', false);
    }
}

async function resetUserBalance(userId) {
    if (!confirm('Reset user balance to ₹1000?')) return;
    
    try {
        await db.collection(USERS_COLLECTION).doc(userId).update({
            balance: 1000,
            lastUpdated: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        showAdminMessage('User balance reset to ₹1000', true);
        
    } catch (error) {
        console.error('Reset balance error:', error);
        showAdminMessage('Error resetting balance', false);
    }
}

async function blockUser(userId) {
    if (!confirm('Block this user? They will not be able to login until unblocked.')) return;
    
    try {
        await db.collection(USERS_COLLECTION).doc(userId).update({
            status: USER_STATUS.BLOCKED,
            blockedAt: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        showAdminMessage('User blocked successfully', true);
        closeUserModal();
        
    } catch (error) {
        console.error('Block user error:', error);
        showAdminMessage('Error blocking user', false);
    }
}

async function unblockUser(userId) {
    try {
        await db.collection(USERS_COLLECTION).doc(userId).update({
            status: USER_STATUS.ACTIVE,
            unblockedAt: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        showAdminMessage('User unblocked successfully', true);
        closeUserModal();
        
    } catch (error) {
        console.error('Unblock user error:', error);
        showAdminMessage('Error unblocking user', false);
    }
}

async function deleteUser(userId) {
    if (!confirm('Delete this user? They can register again with same email.')) return;
    
    try {
        await db.collection(USERS_COLLECTION).doc(userId).update({
            status: USER_STATUS.DELETED,
            balance: 0,
            deletedAt: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        showAdminMessage('User deleted successfully', true);
        closeUserModal();
        
    } catch (error) {
        console.error('Delete user error:', error);
        showAdminMessage('Error deleting user', false);
    }
}

function editUser(userId) {
    viewUserDetails(userId); // For now, use view modal for editing
}

// Bulk Operations
async function bulkDeleteUsers() {
    if (selectedUsers.size === 0) {
        showAdminMessage('Please select users first', false);
        return;
    }
    
    if (!confirm(`Delete ${selectedUsers.size} users? They can register again with same emails.`)) {
        return;
    }
    
    try {
        showAdminLoading(`Deleting ${selectedUsers.size} users...`);
        
        const batch = db.batch();
        selectedUsers.forEach(userId => {
            const userRef = db.collection(USERS_COLLECTION).doc(userId);
            batch.update(userRef, {
                status: USER_STATUS.DELETED,
                balance: 0,
                deletedAt: firebase.firestore.FieldValue.serverTimestamp()
            });
        });
        
        await batch.commit();
        
        hideAdminLoading();
        showAdminMessage(`Successfully deleted ${selectedUsers.size} users`, true);
        selectedUsers.clear();
        document.getElementById('selected-count').textContent = 0;
        
    } catch (error) {
        hideAdminLoading();
        console.error('Bulk delete error:', error);
        showAdminMessage('Error deleting users', false);
    }
}

async function bulkBlockUsers() {
    if (selectedUsers.size === 0) {
        showAdminMessage('Please select users first', false);
        return;
    }
    
    if (!confirm(`Block ${selectedUsers.size} users? They will not be able to login.`)) {
        return;
    }
    
    try {
        showAdminLoading(`Blocking ${selectedUsers.size} users...`);
        
        const batch = db.batch();
        selectedUsers.forEach(userId => {
            const userRef = db.collection(USERS_COLLECTION).doc(userId);
            batch.update(userRef, {
                status: USER_STATUS.BLOCKED,
                blockedAt: firebase.firestore.FieldValue.serverTimestamp()
            });
        });
        
        await batch.commit();
        
        hideAdminLoading();
        showAdminMessage(`Successfully blocked ${selectedUsers.size} users`, true);
        selectedUsers.clear();
        document.getElementById('selected-count').textContent = 0;
        
    } catch (error) {
        hideAdminLoading();
        console.error('Bulk block error:', error);
        showAdminMessage('Error blocking users', false);
    }
}

async function bulkUnblockUsers() {
    if (selectedUsers.size === 0) {
        showAdminMessage('Please select users first', false);
        return;
    }
    
    try {
        showAdminLoading(`Unblocking ${selectedUsers.size} users...`);
        
        const batch = db.batch();
        selectedUsers.forEach(userId => {
            const userRef = db.collection(USERS_COLLECTION).doc(userId);
            batch.update(userRef, {
                status: USER_STATUS.ACTIVE,
                unblockedAt: firebase.firestore.FieldValue.serverTimestamp()
            });
        });
        
        await batch.commit();
        
        hideAdminLoading();
        showAdminMessage(`Successfully unblocked ${selectedUsers.size} users`, true);
        selectedUsers.clear();
        document.getElementById('selected-count').textContent = 0;
        
    } catch (error) {
        hideAdminLoading();
        console.error('Bulk unblock error:', error);
        showAdminMessage('Error unblocking users', false);
    }
}

async function bulkResetBalances() {
    if (selectedUsers.size === 0) {
        showAdminMessage('Please select users first', false);
        return;
    }
    
    if (!confirm(`Reset balances for ${selectedUsers.size} users to ₹1000?`)) {
        return;
    }
    
    try {
        showAdminLoading(`Resetting balances for ${selectedUsers.size} users...`);
        
        const batch = db.batch();
        selectedUsers.forEach(userId => {
            const userRef = db.collection(USERS_COLLECTION).doc(userId);
            batch.update(userRef, {
                balance: 1000,
                lastUpdated: firebase.firestore.FieldValue.serverTimestamp()
            });
        });
        
        await batch.commit();
        
        hideAdminLoading();
        showAdminMessage(`Successfully reset balances for ${selectedUsers.size} users`, true);
        selectedUsers.clear();
        document.getElementById('selected-count').textContent = 0;
        
    } catch (error) {
        hideAdminLoading();
        console.error('Bulk reset balances error:', error);
        showAdminMessage('Error resetting balances', false);
    }
}

// Search Functions
function searchUsers() {
    const searchTerm = document.getElementById('user-search').value.toLowerCase().trim();
    
    if (!searchTerm) {
        loadAllUsers();
        return;
    }
    
    // For now, we'll filter the existing list
    // In a real app, you might want to query Firestore with the search term
    const userItems = document.querySelectorAll('.user-item');
    
    userItems.forEach(item => {
        const userText = item.textContent.toLowerCase();
        if (userText.includes(searchTerm)) {
            item.style.display = 'flex';
        } else {
            item.style.display = 'none';
        }
    });
}

// Transaction Management
async function loadAddMoneyRequests() {
    try {
        const snapshot = await db.collection(ADD_MONEY_COLLECTION)
            .where('status', '==', 'pending')
            .orderBy('requestTime', 'desc')
            .get();
        
        const container = document.getElementById('add-money-requests-list');
        container.innerHTML = '';
        
        if (snapshot.empty) {
            container.innerHTML = '<div class="loading-message">No pending requests</div>';
            return;
        }
        
        snapshot.forEach(doc => {
            const request = doc.data();
            renderAddMoneyRequest(doc.id, request);
        });
        
    } catch (error) {
        console.error('Load add money requests error:', error);
        document.getElementById('add-money-requests-list').innerHTML = 
            '<div class="loading-message">Error loading requests</div>';
    }
}

function renderAddMoneyRequest(requestId, request) {
    const container = document.getElementById('add-money-requests-list');
    
    const requestItem = document.createElement('div');
    requestItem.className = 'request-item';
    requestItem.innerHTML = `
        <div class="request-info">
            <div class="request-user">${request.userName} (${request.userUserId})</div>
            <div class="request-amount">₹${request.amount}</div>
            <div class="request-details">
                Transaction ID: ${request.transactionId}<br>
                Email: ${request.userEmail}
            </div>
            <div class="request-time">
                Requested: ${request.requestTime ? request.requestTime.toDate().toLocaleString() : 'N/A'}
            </div>
        </div>
        <div class="request-actions">
            <button class="approve-btn" onclick="approveAddMoney('${requestId}')">
                Approve
            </button>
            <button class="reject-btn" onclick="rejectAddMoney('${requestId}')">
                Reject
            </button>
        </div>
    `;
    
    container.appendChild(requestItem);
}

async function approveAddMoney(requestId) {
    try {
        const requestDoc = await db.collection(ADD_MONEY_COLLECTION).doc(requestId).get();
        const request = requestDoc.data();
        
        if (!request) {
            showAdminMessage('Request not found', false);
            return;
        }
        
        // Update user balance
        await db.collection(USERS_COLLECTION).doc(request.userId).update({
            balance: firebase.firestore.FieldValue.increment(request.amount),
            lastUpdated: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        // Update request status
        await db.collection(ADD_MONEY_COLLECTION).doc(requestId).update({
            status: 'approved',
            processedAt: firebase.firestore.FieldValue.serverTimestamp(),
            processedBy: 'admin'
        });
        
        showAdminMessage(`Approved ₹${request.amount} for ${request.userName}`, true);
        loadAddMoneyRequests(); // Refresh the list
        
    } catch (error) {
        console.error('Approve add money error:', error);
        showAdminMessage('Error approving request', false);
    }
}

async function rejectAddMoney(requestId) {
    if (!confirm('Reject this add money request?')) return;
    
    try {
        await db.collection(ADD_MONEY_COLLECTION).doc(requestId).update({
            status: 'rejected',
            processedAt: firebase.firestore.FieldValue.serverTimestamp(),
            processedBy: 'admin'
        });
        
        showAdminMessage('Request rejected', true);
        loadAddMoneyRequests(); // Refresh the list
        
    } catch (error) {
        console.error('Reject add money error:', error);
        showAdminMessage('Error rejecting request', false);
    }
}

async function loadWithdrawalRequests() {
    try {
        const snapshot = await db.collection(WITHDRAWAL_COLLECTION)
            .where('status', '==', 'pending')
            .orderBy('requestTime', 'desc')
            .get();
        
        const container = document.getElementById('withdrawal-requests-list');
        container.innerHTML = '';
        
        if (snapshot.empty) {
            container.innerHTML = '<div class="loading-message">No pending requests</div>';
            return;
        }
        
        snapshot.forEach(doc => {
            const request = doc.data();
            renderWithdrawalRequest(doc.id, request);
        });
        
    } catch (error) {
        console.error('Load withdrawal requests error:', error);
        document.getElementById('withdrawal-requests-list').innerHTML = 
            '<div class="loading-message">Error loading requests</div>';
    }
}

function renderWithdrawalRequest(requestId, request) {
    const container = document.getElementById('withdrawal-requests-list');
    
    const requestItem = document.createElement('div');
    requestItem.className = 'request-item';
    requestItem.innerHTML = `
        <div class="request-info">
            <div class="request-user">${request.userName} (${request.userUserId})</div>
            <div class="request-amount">₹${request.amount}</div>
            <div class="request-details">
                Bank: ${request.bankDetails.accountHolder} - ${request.bankDetails.bankName}<br>
                Account: ${request.bankDetails.accountNumber} | IFSC: ${request.bankDetails.ifscCode}<br>
                Email: ${request.userEmail}
            </div>
            <div class="request-time">
                Requested: ${request.requestTime ? request.requestTime.toDate().toLocaleString() : 'N/A'}
            </div>
        </div>
        <div class="request-actions">
            <button class="approve-btn" onclick="approveWithdrawal('${requestId}')">
                Approve
            </button>
            <button class="reject-btn" onclick="rejectWithdrawal('${requestId}')">
                Reject
            </button>
        </div>
    `;
    
    container.appendChild(requestItem);
}

async function approveWithdrawal(requestId) {
    try {
        const requestDoc = await db.collection(WITHDRAWAL_COLLECTION).doc(requestId).get();
        const request = requestDoc.data();
        
        if (!request) {
            showAdminMessage('Request not found', false);
            return;
        }
        
        // Deduct from user balance
        await db.collection(USERS_COLLECTION).doc(request.userId).update({
            balance: firebase.firestore.FieldValue.increment(-request.amount),
            lastUpdated: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        // Update request status
        await db.collection(WITHDRAWAL_COLLECTION).doc(requestId).update({
            status: 'approved',
            processedAt: firebase.firestore.FieldValue.serverTimestamp(),
            processedBy: 'admin'
        });
        
        showAdminMessage(`Approved withdrawal of ₹${request.amount} for ${request.userName}`, true);
        loadWithdrawalRequests(); // Refresh the list
        
    } catch (error) {
        console.error('Approve withdrawal error:', error);
        showAdminMessage('Error approving withdrawal', false);
    }
}

async function rejectWithdrawal(requestId) {
    if (!confirm('Reject this withdrawal request?')) return;
    
    try {
        await db.collection(WITHDRAWAL_COLLECTION).doc(requestId).update({
            status: 'rejected',
            processedAt: firebase.firestore.FieldValue.serverTimestamp(),
            processedBy: 'admin'
        });
        
        showAdminMessage('Withdrawal request rejected', true);
        loadWithdrawalRequests(); // Refresh the list
        
    } catch (error) {
        console.error('Reject withdrawal error:', error);
        showAdminMessage('Error rejecting withdrawal', false);
    }
}

// Analytics Functions
async function loadAnalytics() {
    try {
        // Calculate today's date range
        const today = new Date();
        today.setHours(0, 0, 0, 0);
        const tomorrow = new Date(today);
        tomorrow.setDate(tomorrow.getDate() + 1);
        
        // Get today's transactions
        const [addMoneySnapshot, withdrawalSnapshot, usersSnapshot] = await Promise.all([
            db.collection(ADD_MONEY_COLLECTION)
                .where('requestTime', '>=', firebase.firestore.Timestamp.fromDate(today))
                .where('status', '==', 'approved')
                .get(),
            
            db.collection(WITHDRAWAL_COLLECTION)
                .where('requestTime', '>=', firebase.firestore.Timestamp.fromDate(today))
                .where('status', '==', 'approved')
                .get(),
            
            db.collection(USERS_COLLECTION).get()
        ]);
        
        // Calculate totals
        const totalAddMoney = addMoneySnapshot.docs.reduce((sum, doc) => sum + doc.data().amount, 0);
        const totalWithdrawals = withdrawalSnapshot.docs.reduce((sum, doc) => sum + doc.data().amount, 0);
        const todayProfit = totalAddMoney - totalWithdrawals;
        
        // Update display
        document.getElementById('today-profit').textContent = todayProfit;
        document.getElementById('total-revenue').textContent = totalAddMoney;
        document.getElementById('total-users-count').textContent = usersSnapshot.size;
        
    } catch (error) {
        console.error('Load analytics error:', error);
    }
}

function exportDailyReport() {
    // This would generate and download a report
    // For now, just show a message
    showAdminMessage('Daily report export feature coming soon!', true);
}

// Notification Functions
async function sendNotification() {
    const title = document.getElementById('notification-title').value.trim();
    const message = document.getElementById('notification-message').value.trim();
    const imageUrl = document.getElementById('notification-image').value.trim();
    
    if (!title || !message) {
        showAdminMessage('Please enter title and message', false);
        return;
    }
    
    try {
        await db.collection(NOTIFICATIONS_COLLECTION).add({
            title: title,
            message: message,
            imageUrl: imageUrl || null,
            sentBy: 'admin',
            sentAt: firebase.firestore.FieldValue.serverTimestamp()
        });
        
        // Clear form
        document.getElementById('notification-title').value = '';
        document.getElementById('notification-message').value = '';
        document.getElementById('notification-image').value = '';
        
        showAdminMessage('Notification sent successfully!', true);
        loadNotificationsHistory();
        
    } catch (error) {
        console.error('Send notification error:', error);
        showAdminMessage('Error sending notification', false);
    }
}

async function loadNotificationsHistory() {
    try {
        const snapshot = await db.collection(NOTIFICATIONS_COLLECTION)
            .orderBy('sentAt', 'desc')
            .limit(10)
            .get();
        
        const container = document.getElementById('notifications-history-list');
        container.innerHTML = '';
        
        if (snapshot.empty) {
            container.innerHTML = '<div class="loading-message">No notifications sent yet</div>';
            return;
        }
        
        snapshot.forEach(doc => {
            const notification = doc.data();
            renderNotificationItem(notification);
        });
        
    } catch (error) {
        console.error('Load notifications history error:', error);
        document.getElementById('notifications-history-list').innerHTML = 
            '<div class="loading-message">Error loading notifications</div>';
    }
}

function renderNotificationItem(notification) {
    const container = document.getElementById('notifications-history-list');
    
    const item = document.createElement('div');
    item.className = 'notification-item';
    item.innerHTML = `
        <div class="notification-title">${notification.title}</div>
        <div class="notification-message">${notification.message}</div>
        ${notification.imageUrl ? `<div class="notification-image">Image: ${notification.imageUrl}</div>` : ''}
        <div class="notification-time">
            Sent: ${notification.sentAt ? notification.sentAt.toDate().toLocaleString() : 'N/A'}
        </div>
    `;
    
    container.appendChild(item);
}

// Utility Functions
function showAdminLoading(message = 'Processing...') {
    // You could implement a loading overlay here
    console.log('Loading:', message);
}

function hideAdminLoading() {
    // Hide loading overlay
    console.log('Loading complete');
}

function showAdminMessage(message, isSuccess = true) {
    // You could implement a toast notification system here
    alert(isSuccess ? `✅ ${message}` : `❌ ${message}`);
}

function loadAllData() {
    loadAllUsers();
    loadAddMoneyRequests();
    loadWithdrawalRequests();
    loadAnalytics();
    showAdminMessage('All data refreshed', true);
}

function logoutAdmin() {
    if (confirm('Logout from admin panel?')) {
        document.getElementById('adminContent').classList.remove('active');
        document.getElementById('passwordOverlay').classList.add('active');
        
        // Clear password field
        document.getElementById('adminPassword').value = '';
        document.getElementById('passwordError').style.display = 'none';
        
        // Clean up listeners
        if (gameControlListener) {
            gameControlListener();
            gameControlListener = null;
        }
        
        if (usersListener) {
            usersListener();
            usersListener = null;
        }
        
        // Clear selection
        selectedUsers.clear();
    }
}

// Initialize when page loads
document.addEventListener('DOMContentLoaded', function() {
    // Focus on password input
    document.getElementById('adminPassword').focus();
    
    // Allow Enter key to submit password
    document.getElementById('adminPassword').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') {
            checkAdminPassword();
        }
    });
});
