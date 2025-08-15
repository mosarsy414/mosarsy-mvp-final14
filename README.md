import React, { useState, useEffect } from "react";
import { db, auth, functions } from "./firebaseConfig";
import {
  collection,
  getDocs,
  doc,
  query,
  where,
  orderBy,
  updateDoc,
  addDoc,
  Timestamp,
  setDoc,
  getDoc
} from "firebase/firestore";
import {
  onAuthStateChanged,
  signInWithEmailAndPassword,
  signOut,
  createUserWithEmailAndPassword
} from "firebase/auth";
import { httpsCallable } from "firebase/functions";
import { loadStripe } from '@stripe/stripe-js';
import Web3 from 'web3';
import { Bar } from 'react-chartjs-2';
import { Chart as ChartJS, CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend } from 'chart.js';

ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend);

const stripePromise = loadStripe(pk_test_51RviVkAXApi5lNL7NvYgbmxu5f0DzopLwzGiaSlKxrrXgAtfvSnHvLSeiqIM0Abl977qsabe8xVOnu9pcmraYcIS005N3KCynp); // Replace with Stripe pk_test_...

// BlockchainLogger Component
const BlockchainLogger = ({ action, data }) => {
  const logTransaction = async () => {
    try {
      const web3 = new Web3('https://mainnet.infura.io/v3/your-infura-key'); // Replace with Infura/Alchemy URL
      const accounts = await web3.eth.getAccounts();
      await web3.eth.sendTransaction({
        from: accounts[0],
        to: '0xYourContractAddress', // Replace or mock
        data: web3.utils.toHex(`MÖSARSY Log: ${action} - ${JSON.stringify(data)}`)
      });
      alert('Transaction logged on blockchain!');
    } catch (err) {
      console.log('Mock blockchain log:', action, data);
      alert('Blockchain log mocked (connect MetaMask for real tx)!');
    }
  };
  return <button onClick={logTransaction} className="mt-2 text-blue-600 underline">Log to Blockchain</button>;
};

// RevenueChart Component
const RevenueChart = () => {
  const data = {
    labels: ['Year 1', 'Year 2', 'Year 3'],
    datasets: [
      { label: 'Membership Revenue', data: [200000, 600000, 1800000], backgroundColor: '#1E90FF' },
      { label: 'Platform Usage Fees', data: [600000, 2000000, 6000000], backgroundColor: '#32CD32' },
      { label: 'Loan Interest Share', data: [150000, 500000, 1500000], backgroundColor: '#FFD700' }
    ]
  };
  const options = {
    plugins: { title: { display: true, text: 'MÖSARSY Revenue Projections (Years 1-3)' } },
    scales: { x: { stacked: true }, y: { stacked: true, title: { display: true, text: 'Revenue ($)' } } }
  };
  return <div className="p-4"><Bar data={data} options={options} /></div>;
};

// FinancialCalculator Component
const FinancialCalculator = () => {
  const [members, setMembers] = useState(12);
  const [investment, setInvestment] = useState(100);
  const [duration, setDuration] = useState(12);
  const [interestRate, setInterestRate] = useState(16.82);
  const [position, setPosition] = useState(1);
  const [results, setResults] = useState(null);

  const calculate = () => {
    if (members < 2 || members > 14 || investment < 100 || investment > 1000 || duration < 2 || duration > 14 || interestRate < 1 || interestRate > 25 || position < 1 || position > members) {
      alert('Invalid inputs! Check ROT config limits.');
      return;
    }
    const totalPot = members * investment;
    const loan = (members - position) * investment;
    const platformFee = investment * 0.015;
    const monthlyInterest = (loan * (interestRate / 100)) / duration;
    const monthlyRepay = investment + platformFee + monthlyInterest;
    const rpf = loan * 0.20;
    const avgReturn = 4.59 + (interestRate / 10);
    const profit = (totalPot * (avgReturn / 100)) / members;
    setResults({ totalPot, loan, monthlyRepay: monthlyRepay.toFixed(2), rpf, profit: profit.toFixed(2), avgReturn });
  };

  return (
    <div className="p-4 border rounded">
      <h3 className="text-lg font-semibold mb-2">ROT Financial Calculator</h3>
      <div className="grid grid-cols-2 gap-4">
        <div><label>Members (2-14):</label><input type="number" value={members} onChange={(e) => setMembers(e.target.value)} className="p-2 border w-full" /></div>
        <div><label>Investment ($100-1000):</label><input type="number" value={investment} onChange={(e) => setInvestment(e.target.value)} className="p-2 border w-full" /></div>
        <div><label>Duration (2-14):</label><input type="number" value={duration} onChange={(e) => setDuration(e.target.value)} className="p-2 border w-full" /></div>
        <div><label>Interest Rate (1-25%):</label><input type="number" value={interestRate} onChange={(e) => setInterestRate(e.target.value)} className="p-2 border w-full" /></div>
        <div><label>Your Position (1-{members}):</label><input type="number" value={position} onChange={(e) => setPosition(e.target.value)} className="p-2 border w-full" /></div>
      </div>
      <button onClick={calculate} className="mt-4 bg-blue-600 text-white px-4 py-2 rounded">Calculate</button>
      {results && (
        <div className="mt-4">
          <p>Total Pot: ${results.totalPot}</p>
          <p>Loan for Pos {position}: ${results.loan}</p>
          <p>Monthly Repay: ${results.monthlyRepay}</p>
          <p>RPF (20%): ${results.rpf}</p>
          <p>Est. Profit: ${results.profit} ({results.avgReturn}% return)</p>
          <BlockchainLogger action="ROT Calculated" data={results} />
        </div>
      )}
    </div>
  );
};

// Main App Component
function App() {
  const [user, setUser] = useState(null);
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [isRegistering, setIsRegistering] = useState(false);
  const [error, setError] = useState("");
  const [displayName, setDisplayName] = useState("");
  const [profileSaved, setProfileSaved] = useState(false);
  const [payoutMessages, setPayoutMessages] = useState([]);
  const [acknowledged, setAcknowledged] = useState([]);
  const [inviteEmail, setInviteEmail] = useState("");
  const [invited, setInvited] = useState("");
  const [managedRing, setManagedRing] = useState(null);
  const [members, setMembers] = useState([]);
  const [rings, setRings] = useState([]);
  const [newRingName, setNewRingName] = useState("");
  const [selectedRing, setSelectedRing] = useState("all");
  const [tab, setTab] = useState("dashboard");
  const [savingsData, setSavingsData] = useState({});

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, async (u) => {
      setUser(u);
      if (u) {
        const profileRef = doc(db, "users", u.uid);
        const profileSnap = await getDoc(profileRef);
        if (profileSnap.exists()) {
          const data = profileSnap.data();
          if (data.name) setDisplayName(data.name);
        }
        loadRings();
        loadSavingsData();
      }
    });
    return () => unsubscribe();
  }, []);

  const loadRings = async () => {
    if (!user) return;
    const ringSnapshot = await getDocs(collection(db, "rings"));
    const userRings = [];
    ringSnapshot.forEach((docSnap) => {
      const data = docSnap.data();
      const isMember = data.memberHistory?.some((m) => m.email === user.email);
      if (isMember) {
        userRings.push({ id: docSnap.id, ...data });
        if (data.memberHistory.some((m) => m.email === user.email && m.role === "admin")) {
          setManagedRing({ id: docSnap.id, name: data.ringName });
          setMembers(data.memberHistory);
        }
      }
    });
    setRings(userRings);
  };

  const loadSavingsData = () => {
    setSavingsData({
      investment: 1200,
      loanReceived: 1100,
      monthlyRepay: 116.92,
      profit: 55.01,
      rpf: 220
    });
  };

  const handleLogin = async (e) => {
    e.preventDefault();
    setError("");
    try {
      await signInWithEmailAndPassword(auth, email, password);
    } catch (err) {
      setError("Invalid login credentials");
    }
  };

  const handleRegister = async (e) => {
    e.preventDefault();
    setError("");
    try {
      const cred = await createUserWithEmailAndPassword(auth, email, password);
      await setDoc(doc(db, "users", cred.user.uid), {
        email: cred.user.email,
        createdAt: Timestamp.now()
      });
    } catch (err) {
      setError("Registration failed: " + err.message);
    }
  };

  const handleProfileSave = async () => {
    if (!user) return;
    await updateDoc(doc(db, "users", user.uid), {
      name: displayName,
      updatedAt: Timestamp.now()
    });
    setProfileSaved(true);
    setTimeout(() => setProfileSaved(false), 2000);
  };

  const handleInvite = async () => {
    if (!inviteEmail) return;
    try {
      const sendEmail = httpsCallable(functions, "sendEmailReminder");
      await sendEmail({
        email: inviteEmail,
        subject: "You’re Invited to Join MÖSARSY",
        message: `You've been invited by ${user.email} to join MÖSARSY. Register at [your-app-url]`
      });
      setInvited("Invitation sent to " + inviteEmail);
      setInviteEmail("");
    } catch (err) {
      setError("Failed to send invite");
    }
  };

  const createRing = async () => {
    if (!newRingName || !user) return;
    const newRingRef = await addDoc(collection(db, "rings"), {
      ringName: newRingName,
      memberHistory: [{ email: user.email, role: "admin" }],
      createdAt: Timestamp.now()
    });
    setNewRingName("");
    loadRings();
  };

  const joinRing = async (ringId) => {
    if (!user) return;
    const ringRef = doc(db, "rings", ringId);
    const ringSnap = await getDoc(ringRef);
    if (ringSnap.exists()) {
      const data = ringSnap.data();
      if (data.memberHistory.length < 14) {
        await updateDoc(ringRef, {
          memberHistory: [...data.memberHistory, { email: user.email, role: "member" }]
        });
        loadRings();
      } else {
        setError("Ring is full (max 14 members)");
      }
    }
  };

  useEffect(() => {
    const fetchMessages = async () => {
      if (!user) return;
      const allMessages = [];
      const acknowledgements = [];
      for (const ring of rings) {
        const msgSnap = await getDocs(query(
          collection(db, `rings/${ring.id}/messages`),
          orderBy("timestamp", "desc")
        ));
        msgSnap.forEach((doc) => {
          const msg = doc.data();
          allMessages.push({ ...msg, ringId: ring.id, ringName: ring.ringName, docId: doc.id });
          if (msg.acknowledged?.includes(user.email)) {
            acknowledgements.push(doc.id);
          }
        });
      }
      setPayoutMessages(allMessages);
      setAcknowledged(acknowledgements);
    };
    if (rings.length > 0) fetchMessages();
  }, [rings, user]);

  const handleAcknowledge = async (ringId, docId) => {
    const ref = doc(db, `rings/${ringId}/messages/${docId}`);
    const messageData = payoutMessages.find((m) => m.docId === docId);
    const updatedAck = [...(messageData.acknowledged || []), user.email];
    await updateDoc(ref, { acknowledged: updatedAck });
    setAcknowledged((prev) => [...prev, docId]);

    const ring = rings.find((r) => r.id === ringId);
    const memberEmails = ring?.memberHistory.map((m) => m.email) || [];
    if (memberEmails.every((email) => updatedAck.includes(email))) {
      await addDoc(collection(db, `rings/${ringId}/messages`), {
        sender: "System",
        content: `✅ All members acknowledged payout on ${new Date().toLocaleDateString()}`,
        timestamp: Timestamp.now(),
        acknowledged: []
      });
    }
  };

  const promoteMember = async (email) => {
    const updated = members.map((m) => (m.email === email ? { ...m, role: "admin" } : m));
    await updateDoc(doc(db, "rings", managedRing.id), { memberHistory: updated });
    setMembers(updated);
  };

  const demoteMember = async (email) => {
    const updated = members.map((m) => (m.email === email ? { ...m, role: "member" } : m));
    await updateDoc(doc(db, "rings", managedRing.id), { memberHistory: updated });
    setMembers(updated);
  };

  const removeMember = async (email) => {
    const updated = members.filter((m) => m.email !== email);
    await updateDoc(doc(db, "rings", managedRing.id), { memberHistory: updated });
    setMembers(updated);
  };

  const handlePayment = async () => {
    const stripe = await stripePromise;
    try {
      const response = await fetch('/create-checkout-session', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount: 2000 })
      });
      const { id } = await response.json();
      const { error } = await stripe.redirectToCheckout({ sessionId: id });
      if (error) setError(error.message);
    } catch (err) {
      setError("Payment failed");
    }
  };

  if (!user) {
    return (
      <div className="p-8 max-w-sm mx-auto mt-20 border rounded shadow bg-white">
        <h2 className="text-xl font-bold mb-4">{isRegistering ? "Register" : "Login"} to MÖSARSY</h2>
        <form onSubmit={isRegistering ? handleRegister : handleLogin}>
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            placeholder="Email"
            className="w-full p-2 border mb-2 rounded"
            required
          />
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            placeholder="Password"
            className="w-full p-2 border mb-4 rounded"
            required
          />
          <button type="submit" className="w-full bg-blue-600 text-white py-2 rounded">
            {isRegistering ? "Create Account" : "Log In"}
          </button>
          {error && <p className="text-red-500 mt-2 text-sm">{error}</p>}
        </form>
        <p className="mt-4 text-sm text-center">
          {isRegistering ? "Already have an account?" : "Need an account?"}{" "}
          <button onClick={() => setIsRegistering(!isRegistering)} className="text-blue-600 underline">
            {isRegistering ? "Log In" : "Register"}
          </button>
        </p>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-100">
      <div className="p-8 max-w-4xl mx-auto bg-white rounded shadow">
        <div className="flex justify-between items-center mb-6">
          <div>
            <h2 className="text-2xl font-bold">Welcome, {displayName || user.email}</h2>
            <p className="text-sm text-gray-600">Your fintech partner in community savings!</p>
          </div>
          <button onClick={() => signOut(auth)} className="text-sm text-red-500 underline">
            Log out
          </button>
        </div>

        <div className="flex space-x-4 mb-6">
          <button onClick={() => setTab("dashboard")} className={`px-4 py-2 rounded ${tab === "dashboard" ? "bg-blue-600 text-white" : "bg-gray-200"}`}>Dashboard</button>
          <button onClick={() => setTab("rings")} className={`px-4 py-2 rounded ${tab === "rings" ? "bg-blue-600 text-white" : "bg-gray-200"}`}>Rings</button>
          <button onClick={() => setTab("payouts")} className={`px-4 py-2 rounded ${tab === "payouts" ? "bg-blue-600 text-white" : "bg-gray-200"}`}>Payouts</button>
          <button onClick={() => setTab("profile")} className={`px-4 py-2 rounded ${tab === "profile" ? "bg-blue-600 text-white" : "bg-gray-200"}`}>Profile</button>
          <button onClick={() => setTab("calculator")} className={`px-4 py-2 rounded ${tab === "calculator" ? "bg-blue-600 text-white" : "bg-gray-200"}`}>Calculator</button>
          <button onClick={() => setTab("revenue")} className={`px-4 py-2 rounded ${tab === "revenue" ? "bg-blue-600 text-white" : "bg-gray-200"}`}>Revenue</button>
        </div>

        {tab === "dashboard" && (
          <div>
            <h3 className="text-lg font-semibold mb-2">Your Savings Overview</h3>
            <div className="grid grid-cols-2 gap-4">
              <div className="p-4 border rounded">
                <p>Investment: ${savingsData.investment}</p>
                <p>Loan Received: ${savingsData.loanReceived}</p>
              </div>
              <div className="p-4 border rounded">
                <p>Monthly Repay: ${savingsData.monthlyRepay?.toFixed(2)}</p>
                <p>Profit Earned: ${savingsData.profit?.toFixed(2)} (4.59% return)</p>
              </div>
              <div className="p-4 border rounded col-span-2">
                <p>Risk Provision Fund (RPF): ${savingsData.rpf} (20% protected)</p>
              </div>
            </div>
          </div>
        )}

        {tab === "rings" && (
          <div>
            <h3 className="text-lg font-semibold mb-2">Your Rings of Trust</h3>
            <ul className="list-disc ml-6 mb-4">
              {rings.map((ring) => (
                <li key={ring.id}>
                  {ring.ringName} (Members: {ring.memberHistory.length}/14)
                  {!ring.memberHistory.some((m) => m.email === user.email) && (
                    <button onClick={() => joinRing(ring.id)} className="ml-2 text-green-600 underline">Join</button>
                  )}
                </li>
              ))}
            </ul>
            {managedRing && (
              <div className="mb-4">
                <h4 className="font-semibold">Manage {managedRing.name}</h4>
                <ul className="list-disc ml-6">
                  {members.map((m, idx) => (
                    <li key={idx}>
                      {m.email} — {m.role}
                      {m.email !== user.email && (
                        <>
                          {m.role === "member" ? (
                            <button onClick={() => promoteMember(m.email)} className="ml-2 text-green-600 underline">Promote</button>
                          ) : (
                            <button onClick={() => demoteMember(m.email)} className="ml-2 text-yellow-600 underline">Demote</button>
                          )}
                          <button onClick={() => removeMember(m.email)} className="ml-2 text-red-600 underline">Remove</button>
                        </>
                      )}
                    </li>
                  ))}
                </ul>
              </div>
            )}
            <div>
              <input
                type="text"
                value={newRingName}
                onChange={(e) => setNewRingName(e.target.value)}
                placeholder="New Ring Name"
                className="p-2 border rounded mr-2"
              />
              <button onClick={createRing} className="bg-blue-600 text-white px-4 py-2 rounded">
                Create Ring
              </button>
            </div>
          </div>
        )}

        {tab === "payouts" && (
          <div>
            <h3 className="text-lg font-semibold mb-2">Payout History</h3>
            <select value={selectedRing} onChange={(e) => setSelectedRing(e.target.value)} className="mb-4 p-2 border rounded block">
              <option value="all">All Rings</option>
              {rings.map((ring) => <option key={ring.id} value={ring.id}>{ring.ringName}</option>)}
            </select>
            <ul className="list-disc ml-6">
              {payoutMessages
                .filter((msg) => selectedRing === "all" || msg.ringId === selectedRing)
                .map((msg, i) => (
                  <li key={i} className="mb-2">
                    <strong>{msg.ringName}</strong>: {msg.content} — {msg.timestamp?.toDate().toLocaleString()}
                    {!acknowledged.includes(msg.docId) && (
                      <button onClick={() => handleAcknowledge(msg.ringId, msg.docId)} className="ml-4 text-blue-600 underline text-sm">
                        Acknowledge
                      </button>
                    )}
                    {acknowledged.includes(msg.docId) && <span className="ml-2 text-green-600 text-sm">✓ Acknowledged</span>}
                  </li>
                ))}
              {payoutMessages.length === 0 && <li>No payouts yet</li>}
            </ul>
          </div>
        )}

        {tab === "profile" && (
          <div>
            <h3 className="text-lg font-semibold mb-2">Edit Profile</h3>
            <input
              type="text"
              value={displayName}
              onChange={(e) => setDisplayName(e.target.value)}
              placeholder="Full Name"
              className="p-2 border rounded w-full mb-2"
            />
            <button onClick={handleProfileSave} className="bg-blue-600 text-white px-4 py-2 rounded">
              Save
            </button>
            {profileSaved && <p className="text-green-600 mt-2">✅ Updated!</p>}
            <div className="mt-4">
              <input
                type="email"
                value={inviteEmail}
                onChange={(e) => setInviteEmail(e.target.value)}
                placeholder="Invite by email"
                className="p-2 border rounded mr-2"
              />
              <button onClick={handleInvite} className="bg-green-600 text-white px-4 py-2 rounded">
                Invite
              </button>
              {invited && <p className="mt-2 text-green-600">{invited}</p>}
              <button onClick={handlePayment} className="bg-green-600 text-white px-4 py-2 rounded mt-4">
                Pay $20 Membership
              </button>
            </div>
            <BlockchainLogger action="Profile Updated" data={{ user: user.email, name: displayName }} />
          </div>
        )}

        {tab === "calculator" && <FinancialCalculator />}
        {tab === "revenue" && <RevenueChart />}
        {error && <p className="text-red-500 mt-4">{error}</p>}
      </div>
    </div>
  );
}

export default App;
