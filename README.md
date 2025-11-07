<!DOCTYPE html>
<html lang="bn">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RM71 Gaming Portal</title>
    <!-- Tailwind CSS লোড করা হচ্ছে -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* ইন্টার ফন্ট ব্যবহার করা হচ্ছে */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #1a1a2e; /* গাঢ় নীল/বেগুনি ব্যাকগ্রাউন্ড */
            color: #ffffff;
        }
        .main-container {
            min-height: calc(100vh - 80px); /* ফুটারের জন্য জায়গা রাখা */
            padding-bottom: 80px;
        }
        /* কাস্টম স্ক্রলবার */
        ::-webkit-scrollbar {
            width: 8px;
        }
        ::-webkit-scrollbar-thumb {
            background: #28a745;
            border-radius: 4px;
        }
        ::-webkit-scrollbar-track {
            background: #1a1a2e;
        }
    </style>
</head>
<body>

    <!-- FIREBASE এবং AUTH ইম্পোর্ট করা হলো (এই সেকশনটি আর কাজ করবে না লোকাল ফাইলে, কিন্তু স্ট্রাকচার রাখা হলো) -->
    <script type="module">
        // লোকালি ভিউ করার জন্য এই অংশগুলি ব্লক করা হলো
        /* import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, onSnapshot, setDoc, runTransaction, collection, query, where, getDocs, updateDoc, writeBatch } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        */

        // ফায়ারবেস ফাংশনগুলিকে মক করা হলো যাতে কোড চলতে পারে
        window.RM71 = {
            db: null, // মকড
            auth: null, // মকড
            appId: 'default-app-id',
            userId: 'LocalUser12345',
            user: {
                points: 150, // উদাহরণ হিসেবে কিছু পয়েন্ট দেওয়া হলো
                diamonds: 50, // উদাহরণ হিসেবে কিছু ডায়মন্ড দেওয়া হলো
                referralCount: 15,
                vipLevel: 1,
                name: "ব্যবহারকারী",
                uid: 'LocalUser12345'
            },
            currentPage: 'home',
            isAuthReady: true, // এখন আমরা ধরে নিচ্ছি এটি রেডি
            message: { type: '', text: '' },
            giftCodes: {
                today: "RM71TODAY20",
                weekly: "RM71WEEKLY100"
            }
        };

        // --- Core Functions ---
        
        // সেকশন পরিবর্তন
        window.navigateTo = (page) => {
            RM71.currentPage = page;
            renderApp();
        };

        // কাস্টম মেসেজ দেখানো (ফাংশন কাজ করবে, কিন্তু ডাটাবেস আপডেট হবে না)
        window.showMessage = (text, type = 'success', duration = 4000) => {
            RM71.message = { text, type };
            renderApp();
            setTimeout(() => {
                RM71.message = { type: '', text: '' };
                renderApp();
            }, duration);
        };

        // রেন্ডারিং ফাংশন
        window.renderApp = () => {
            const container = document.getElementById('main-content');
            if (!container) return;

            // হেডার (পয়েন্ট/ডায়মন্ড ব্যালেন্স) আপডেট
            document.getElementById('points-balance').textContent = RM71.user.points.toLocaleString('bn-BD');
            document.getElementById('diamond-balance').textContent = RM71.user.diamonds.toLocaleString('bn-BD');
            document.getElementById('user-uid-display').textContent = RM71.user.uid || 'N/A';
            
            // পেইজের বিষয়বস্তু রেন্ডার করা
            container.innerHTML = getPageContent(RM71.currentPage);
            
            // মেসেজ বক্স রেন্ডার করা
            renderMessage();
            
            // ইভেন্ট লিসেনার সেট করা
            attachEventListeners();
        };

        // কাস্টম মেসেজ বক্স রেন্ডারিং
        window.renderMessage = () => {
            const msgBox = document.getElementById('message-box');
            if (!msgBox) return;

            const { text, type } = RM71.message;
            if (!text) {
                msgBox.innerHTML = '';
                return;
            }

            let bgColor, icon;
            if (type === 'success') {
                bgColor = 'bg-green-500';
                icon = '✅';
            } else if (type === 'error') {
                bgColor = 'bg-red-500';
                icon = '❌';
            } else {
                bgColor = 'bg-yellow-500';
                icon = 'ℹ️';
            }

            msgBox.innerHTML = `
                <div class="fixed top-20 right-4 p-4 rounded-lg shadow-xl z-50 ${bgColor} text-white transition-opacity duration-300">
                    <div class="flex items-center">
                        <span class="text-xl mr-2">${icon}</span>
                        <p class="font-semibold">${text}</p>
                    </div>
                </div>
            `;
        };

        // --- Event Handlers (Handlers are defined here) ---
        
        // ফাংশন ৪: পাসকি জেনারেটর ফর্ম হ্যান্ডেল (এটি এখন শুধুমাত্র UI দেখাবে, ডাটা সেভ করবে না)
        window.handlePasskeyGenerator = async (event) => {
            event.preventDefault();
            const form = event.target;
            const name = form.name.value.trim();
            const email = form.email.value.trim();
            const nationality = form.nationality.value.trim();

            if (!name || !email || !nationality) {
                showMessage("সব ঘর পূরণ করুন।", 'error');
                return;
            }
            
            const passkey = `${name.substring(0, 3).toUpperCase()}-${Math.random().toString(36).substring(2, 8).toUpperCase()}-${RM71.userId.substring(0, 4).toUpperCase()}`;
            
            showMessage(`পাসকি জেনারেট হয়েছে (ডাটা সেভ হবে না): ${passkey}`, 'success', 8000);
            document.getElementById('passkey-result').innerHTML = `
                <p class="text-xl font-bold text-green-400 mt-4">আপনার পাসকি:</p>
                <p class="text-3xl bg-gray-800 p-3 rounded-lg mt-1 select-all">${passkey}</p>
                <p class="text-sm text-gray-400 mt-2">আপনার পাসকি নিরাপদে সংরক্ষণ করুন। (ডাটাবেস মকড)</p>
            `;
        };

        // অন্যান্য ফাংশন যেমন: earnPointByAd, handleBuyDiamond, handlePointRedeem, handleGiftCodeRedeem, handleVIPCheck, handleReportSubmit 
        // এগুলি সব এখন ডাটাবেসে সেভ না করে শুধুমাত্র একটি সফল মেসেজ দেখাবে।

        window.earnPointByAd = async () => { showMessage("অভিনন্দন! আপনি ১ পয়েন্ট অর্জন করেছেন (ডাটাবেস মকড)।", 'success'); };
        window.handleBuyDiamond = async (event) => { event.preventDefault(); showMessage("লেনদেন সফলভাবে জমা দেওয়া হয়েছে (ডাটাবেস মকড)।", 'success', 8000); };
        window.handlePointRedeem = async (event) => { 
            event.preventDefault();
            const form = event.target;
            const packageId = form.redeemPackage.value;
            const packageDetails = { 'redeem1': { pointsRequired: 209 }, 'redeem2': { pointsRequired: 709 } }[packageId];
            if (RM71.user.points < (packageDetails?.pointsRequired || Infinity)) {
                 showMessage(`এই প্যাকেজের জন্য আপনার যথেষ্ট পয়েন্ট নেই।`, 'error');
                 return;
            }
            showMessage(`সফলভাবে রিডিম করা হয়েছে! (ডাটাবেস মকড)`, 'success', 8000);
            form.reset();
        };
        window.handleGiftCodeRedeem = async (event) => { event.preventDefault(); showMessage("গিফট কোড সফলভাবে জমা দেওয়া হয়েছে (ডাটাবেস মকড)।", 'success', 8000); };
        window.handleVIPCheck = async (event) => { event.preventDefault(); showMessage("ভিআইপি স্ট্যাটাস আপডেট হয়েছে (ডাটাবেস মকড)।", 'success', 8000); };
        window.handleReportSubmit = async (event) => { event.preventDefault(); showMessage("রিপোর্ট সফলভাবে জমা দেওয়া হয়েছে (ডাটাবেস মকড)।", 'success'); };


        // --- UI Rendering Functions (HTML Templates) ---
        
        // সব সেকশনের জন্য একটি প্রধান ফাংশন
        window.getPageContent = (page) => {
            // (এখানে সকল সেকশন রেন্ডার করার ফাংশন আছে, যা পরিবর্তন করা হয়নি)
            switch (page) {
                case 'about': return renderAboutSection();
                case 'downloads': return renderDownloadsSection();
                case 'passkey': return renderPasskeySection();
                case 'giftcode': return renderGiftCodeSection();
                case 'events': return renderEventsSection();
                case 'referral': return renderReferralSection();
                case 'buy_diamond': return renderBuyDiamondSection();
                case 'redeem': return renderRedeemSection();
                case 'vip': return renderVIPSection();
                case 'report': return renderReportSection();
                case 'owner': return renderOwnerSection();
                default: return renderHomeSection();
            }
        };

        // (বাকি সমস্ত render* এবং helper ফাংশন আগের মতোই কাজ করবে)
        window.renderHomeSection = () => `
            <div class="p-6 md:p-10 text-center">
                <h2 class="text-5xl font-extrabold text-green-400 mb-4 animate-pulse">RM71 গেমিং পোর্টাল</h2>
                <p class="text-xl text-gray-300 mb-8">সবথেকে সেরা গেমিং অভিজ্ঞতা পেতে RM71-এর সাথে যুক্ত থাকুন!</p>
                <div class="grid md:grid-cols-3 gap-6">
                    ${renderFeatureCard("পয়েন্ট আর্ন", "বিজ্ঞাপন দেখুন বা গেম খেলে পয়েন্ট অর্জন করুন।", "referral")}
                    ${renderFeatureCard("ডায়মন্ড কিনুন", "বিশেষ প্যাকেজে আপনার প্রয়োজনীয় ডায়মন্ড কিনুন।", "buy_diamond")}
                    ${renderFeatureCard("ভিআইপি স্ট্যাটাস", "রেফার করে ভিআইপি লেভেলে আপগ্রেড করুন ও দৈনিক ডায়মন্ড পান।", "vip")}
                </div>
            </div>
        `;
        
        window.renderAboutSection = () => `
            <div class="p-6 md:p-10">
                <h2 class="text-4xl font-bold text-green-400 mb-6">RM71 টিম সম্পর্কে</h2>
                <p class="text-gray-300 mb-6">RM71 হল একটি প্যাশনেট ডেভেলপার ও গেমিং টিম যারা ব্যবহারকারীদের জন্য সেরা অ্যাপ এবং গেমিং অভিজ্ঞতা দিতে প্রতিশ্রুতিবদ্ধ। আমাদের লক্ষ্য হল নতুন ও উত্তেজনাপূর্ণ ডিজিটাল প্রোডাক্ট তৈরি করা।</p>
                ${renderTeamMembers()}
            </div>
        `;

        window.renderDownloadsSection = () => `
            <div class="p-6 md:p-10 text-center">
                <h2 class="text-4xl font-bold text-green-400 mb-8">অ্যাপ এবং গেম ডাউনলোড</h2>
                <div class="flex flex-col items-center space-y-6">
                    ${renderDownloadButton("RM71 অফিসিয়াল অ্যাপ", "https://example.com/rm71app", "bg-blue-600")}
                    ${renderDownloadButton("RM71 ফাস্ট গেম", "https://example.com/rm71game1", "bg-purple-600")}
                    ${renderDownloadButton("RM71 পাজল গেম", "https://example.com/rm71game2", "bg-red-600")}
                </div>
                <p class="text-sm text-gray-400 mt-8">সমস্ত ডাউনলোড লিঙ্ক Google Play Store/Official Website-এর দিকে নির্দেশ করে।</p>
            </div>
        `;

        window.renderPasskeySection = () => `
            <div class="p-6 md:p-10 max-w-lg mx-auto bg-[#24243e] rounded-xl shadow-2xl">
                <h2 class="text-4xl font-bold text-green-400 mb-6 text-center">পাসকি জেনারেটর</h2>
                <form id="passkey-form" onsubmit="handlePasskeyGenerator(event)">
                    <div class="mb-4">
                        <label for="name" class="block mb-2 text-sm font-medium text-gray-300">১. আপনার নাম লিখুন</label>
                        <input type="text" id="name" name="name" required class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white">
                    </div>
                    <div class="mb-4">
                        <label for="email" class="block mb-2 text-sm font-medium text-gray-300">২. ইমেইল লিখুন</label>
                        <input type="email" id="email" name="email" required class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white">
                    </div>
                    <div class="mb-6">
                        <label for="nationality" class="block mb-2 text-sm font-medium text-gray-300">৩. জাতীয়তা</label>
                        <input type="text" id="nationality" name="nationality" required class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white">
                    </div>
                    
                    <button type="submit" class="w-full bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-4 rounded-lg transition duration-200">
                        পাসকি জেনারেট করুন
                    </button>
                </form>
                <div id="passkey-result" class="mt-6 text-center">
                    <!-- জেনারেট করা পাসকি এখানে দেখা যাবে -->
                </div>
            </div>
        `;

        window.renderGiftCodeSection = () => `
            <div class="p-6 md:p-10">
                <h2 class="text-4xl font-bold text-green-400 mb-6 text-center">গিফট কোড</h2>
                <p class="text-lg text-gray-400 text-center mb-8">এই কোডগুলো ব্যবহার করে গেমে ফ্রি ডায়মন্ড বা গিফট জিতে নিন!</p>
                <div class="grid md:grid-cols-2 gap-6 max-w-3xl mx-auto">
                    ${renderCodeCard("আজকের গিফট কোড", RM71.giftCodes.today, "bg-indigo-600")}
                    ${renderCodeCard("সাপ্তাহিক গিফট কোড", RM71.giftCodes.weekly, "bg-orange-600")}
                </div>
                
                <h3 class="text-2xl font-semibold text-gray-200 mt-12 mb-4 text-center">১১. গিফট কোড রিডিম অপশন</h3>
                ${renderGiftCodeRedeemForm()}
            </div>
        `;

        window.renderGiftCodeRedeemForm = () => `
            <div class="max-w-md mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl">
                <form id="gift-code-redeem-form" onsubmit="handleGiftCodeRedeem(event)">
                    <div class="mb-4">
                        <label for="uid" class="block mb-2 text-sm font-medium text-gray-300">আপনার UID লিখুন</label>
                        <input type="text" id="uid" name="uid" required value="${RM71.user.uid || ''}" class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white" placeholder="আপনার গেম ID">
                    </div>
                    <div class="mb-6">
                        <label for="giftCode" class="block mb-2 text-sm font-medium text-gray-300">গিফট কোড লিখুন</label>
                        <input type="text" id="giftCode" name="giftCode" required class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white" placeholder="যেমন: GIFT-XXXXXXXX">
                    </div>
                    <button type="submit" class="w-full bg-red-500 hover:bg-red-600 text-white font-bold py-3 px-4 rounded-lg transition duration-200">
                        সাবমিট করুন
                    </button>
                </form>
                <p class="text-sm text-center text-gray-400 mt-4">সাবমিট করার ২ দিনের মধ্যে আপনার উপহার পেয়ে যাবেন। (ডাটাবেস মকড)</p>
            </div>
        `;

        window.renderEventsSection = () => `
            <div class="p-6 md:p-10">
                <h2 class="text-4xl font-bold text-green-400 mb-6 text-center">বিশেষ ইভেন্টসমূহ</h2>
                <div class="space-y-6 max-w-2xl mx-auto">
                    ${renderEventCard("গ্র্যান্ড রিলিজ ইভেন্ট", "আমাদের নতুন গেম লঞ্চ উপলক্ষে বিশেষ পুরস্কার, ডাবল পয়েন্ট এবং এক্সক্লুসিভ আইটেম জিতে নিন! চলবে: ২৯শে নভেম্বর পর্যন্ত।", "25 Nov 2025")}
                    ${renderEventCard("উইকলি টুর্নামেন্ট", "প্রতি শুক্রবার রাত ৯টায় টুর্নামেন্টে যোগ দিন এবং ১০০০ ডায়মন্ডের পুরস্কার জিতে নিন।", "Every Friday")}
                    <p class="text-gray-400 text-center pt-4">আরো ইভেন্টের জন্য নিয়মিত ভিজিট করুন!</p>
                </div>
            </div>
        `;

        window.renderReferralSection = () => `
            <div class="p-6 md:p-10">
                <h2 class="text-4xl font-bold text-green-400 mb-8 text-center">রেফার করুন ও পয়েন্ট জিতুন</h2>
                
                <div class="grid md:grid-cols-3 gap-6 mb-8 max-w-4xl mx-auto">
                    ${renderStatCard("পয়েন্ট ব্যালেন্স", RM71.user.points.toLocaleString('bn-BD') + " পয়েন্ট", "text-yellow-400")}
                    ${renderStatCard("রেফারেল সংখ্যা", RM71.user.referralCount.toLocaleString('bn-BD') + " জন", "text-green-400")}
                    ${renderStatCard("প্রতি রেফারে", "১০ পয়েন্ট", "text-blue-400")}
                </div>

                <div class="max-w-xl mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl mb-8">
                    <h3 class="text-2xl font-semibold mb-4 text-center">আপনার ইনভাইট লিংক</h3>
                    <div class="relative mb-4">
                        <input type="text" id="invite-link" value="https://rm71.com/invite?ref=${RM71.userId}" readonly class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 text-white select-all">
                        <button onclick="copyInviteLink()" class="absolute right-0 top-0 h-full px-4 bg-green-500 hover:bg-green-600 rounded-r-lg transition duration-200">
                            কপি
                        </button>
                    </div>
                    <p class="text-sm text-gray-400 text-center">এই লিংকটি শেয়ার করুন। ১ জন ডাউনলোড করলে আপনি ১০ পয়েন্ট পাবেন।</p>
                </div>

                <!-- সেকশন ৮: বিজ্ঞাপন দেখে পয়েন্ট অর্জন -->
                <div class="max-w-xl mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl mt-12 text-center">
                    <h3 class="text-2xl font-semibold mb-4">৮. বিজ্ঞাপন দেখে পয়েন্ট অর্জন করুন</h3>
                    <p class="text-lg text-gray-400 mb-4">প্রতিটি বিজ্ঞাপনের জন্য ১ পয়েন্ট।</p>
                    <button onclick="earnPointByAd()" class="bg-indigo-500 hover:bg-indigo-600 text-white font-bold py-3 px-8 rounded-lg transition duration-200 text-xl">
                        বিজ্ঞাপন দেখুন (১ পয়েন্ট)
                    </button>
                </div>
            </div>
        `;

        window.renderBuyDiamondSection = () => {
            const packages = [
                { id: 'pkg1', bdt: 100, diamonds: 125 },
                { id: 'pkg2', bdt: 159, diamonds: 192 },
                { id: 'pkg3', bdt: 500, diamonds: 714 },
                { id: 'pkg4', bdt: 1000, diamonds: 1470 }
            ];

            return `
                <div class="p-6 md:p-10">
                    <h2 class="text-4xl font-bold text-green-400 mb-8 text-center">৯. ডায়মন্ড বা পয়েন্ট কিনুন</h2>
                    <div class="text-center text-gray-300 mb-8">
                        <p class="text-lg font-semibold">১ ডায়মন্ড = ১ টাকা (BDT)</p>
                        <p class="text-sm">সর্বনিম্ন ২০ ডায়মন্ড কিনতে হবে।</p>
                    </div>

                    <div class="grid md:grid-cols-4 gap-6 max-w-5xl mx-auto mb-10">
                        ${packages.map(p => `
                            <label class="block cursor-pointer">
                                <input type="radio" name="package-select-mock" value="${p.id}" class="hidden peer">
                                <div class="bg-[#24243e] p-5 rounded-xl border-4 border-transparent peer-checked:border-green-500 transition duration-200 hover:shadow-lg hover:shadow-green-500/30">
                                    <p class="text-3xl font-extrabold text-yellow-400">${p.diamonds}</p>
                                    <p class="text-sm text-gray-400">ডায়মন্ড</p>
                                    <p class="text-xl font-bold mt-2">${p.bdt} BDT</p>
                                </div>
                            </label>
                        `).join('')}
                    </div>

                    <div class="max-w-xl mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl">
                        <h3 class="text-2xl font-semibold mb-4 text-center">পেমেন্ট এবং ট্রানজেকশন</h3>
                        <div class="bg-red-900/50 p-4 rounded-lg mb-6 text-sm">
                            <p class="font-bold mb-2 text-red-300">⚠️ শুধুমাত্র 'সেন্ড মানি' (Send Money) করুন। অন্যথায় ডায়মন্ড পাবেন না।</p>
                        </div>
                        
                        <form id="buy-diamond-form" onsubmit="handleBuyDiamond(event)">
                            <div class="mb-4">
                                <label for="paymentMethod" class="block mb-2 text-sm font-medium text-gray-300">পেমেন্ট মেথড নির্বাচন করুন</label>
                                <select id="paymentMethod" name="paymentMethod" required class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white">
                                    <option value="">নির্বাচন করুন</option>
                                    <option value="bikash">বিকাশ (bKash): 01620382203</option>
                                    <option value="nogod">নগদ (Nagad): 01828941805</option>
                                </select>
                            </div>
                            <div class="mb-4">
                                <label for="tnxId" class="block mb-2 text-sm font-medium text-gray-300">আপনার Trx ID লিখুন</label>
                                <input type="text" id="tnxId" name="tnxId" required class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white" placeholder="যেমন: 8S1A5J9K4">
                            </div>
                            <!-- প্যাকেজ আইডি লুকিয়ে রাখা হলো (JavaScript-এর জন্য) -->
                            <input type="hidden" id="package" name="package" required value="">

                            <button type="submit" class="w-full bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-4 rounded-lg transition duration-200 mt-4">
                                ডায়মন্ড কিনুন (ট্রানজেকশন জমা দিন)
                            </button>
                        </form>
                        <p class="text-sm text-center text-gray-400 mt-4">ডায়মন্ড পেতে ১ দিন সময় লাগতে পারে। (ডাটাবেস মকড)</p>
                    </div>
                </div>
            `;
        };

        window.renderRedeemSection = () => {
            const redeemPackages = [
                { id: 'redeem1', points: 209, gift: '১০০ ডায়মন্ড + র্যান্ডম গিফট' },
                { id: 'redeem2', points: 709, gift: '৫০০ ডায়মন্ড + র্যান্ডম ইমোট' }
            ];

            return `
                <div class="p-6 md:p-10">
                    <h2 class="text-4xl font-bold text-green-400 mb-8 text-center">১০. পয়েন্ট রিডিম অপশন</h2>
                    <div class="grid md:grid-cols-2 gap-6 max-w-3xl mx-auto mb-8">
                        ${redeemPackages.map(p => `
                            <div class="bg-[#24243e] p-6 rounded-xl shadow-2xl text-center">
                                <p class="text-3xl font-extrabold text-red-400">${p.points}</p>
                                <p class="text-sm text-gray-400 mb-3">পয়েন্ট প্রয়োজন</p>
                                <h3 class="text-xl font-semibold text-white mb-4">${p.gift}</h3>
                                <button onclick="selectRedeemPackage('${p.id}')" class="redeem-select-btn bg-yellow-500 hover:bg-yellow-600 text-gray-900 font-bold py-2 px-4 rounded-lg transition duration-200">
                                    প্যাকেজটি নির্বাচন করুন
                                </button>
                            </div>
                        `).join('')}
                    </div>

                    <div class="max-w-md mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl mt-10">
                        <h3 class="text-2xl font-semibold mb-4 text-center">রিডিম করুন</h3>
                        <form id="point-redeem-form" onsubmit="handlePointRedeem(event)">
                            <input type="hidden" id="redeemPackage" name="redeemPackage" required value="">
                            <p id="redeem-status" class="text-center text-red-400 mb-4 font-semibold">
                                অনুগ্রহ করে রিডিম করার জন্য একটি প্যাকেজ নির্বাচন করুন।
                            </p>
                            <button type="submit" id="redeem-submit-btn" disabled class="w-full bg-gray-500 text-white font-bold py-3 px-4 rounded-lg transition duration-200 cursor-not-allowed">
                                রিডিম করুন
                            </button>
                        </form>
                    </div>
                </div>
            `;
        };

        window.renderVIPSection = () => {
            const vipLevels = [
                { level: 1, referrals: 50, reward: "দৈনিক ৫০ ডায়মন্ড" },
                { level: 2, referrals: 100, reward: "দৈনিক ১০০ ডায়মন্ড" },
                { level: 3, referrals: 1000, reward: "দৈনিক ১০০০ ডায়মন্ড (Max)" }
            ];
            const currentLevel = RM71.user.vipLevel;

            return `
                <div class="p-6 md:p-10">
                    <h2 class="text-4xl font-bold text-green-400 mb-6 text-center">১২. ভিআইপি মেম্বারশিপ</h2>
                    <div class="max-w-4xl mx-auto mb-8 bg-[#24243e] p-6 rounded-xl shadow-2xl">
                        <h3 class="text-2xl font-semibold mb-4 text-center">আপনার বর্তমান স্ট্যাটাস:</h3>
                        <div class="text-center">
                            <p class="text-lg text-gray-300">আপনার বর্তমান রেফারেল সংখ্যা: <span class="text-green-400 font-bold">${RM71.user.referralCount}</span> জন</p>
                            <p class="text-lg text-gray-300">বর্তমান লেভেল: <span class="text-yellow-400 font-bold">লেভেল ${currentLevel}</span></p>
                            <p class="text-lg text-gray-300">দৈনিক পুরস্কার: <span class="text-blue-400 font-bold">${currentLevel === 1 ? '৫০' : currentLevel === 2 ? '১০০' : currentLevel === 3 ? '১০০০' : '০'} ডায়মন্ড</span></p>
                        </div>
                    </div>

                    <h3 class="text-2xl font-semibold text-gray-200 mt-12 mb-6 text-center">ভিআইপি লেভেলসমূহ</h3>
                    <div class="grid md:grid-cols-3 gap-6 max-w-4xl mx-auto">
                        ${vipLevels.map(v => `
                            <div class="p-5 rounded-xl text-center shadow-lg ${v.level === currentLevel ? 'bg-yellow-800 border-4 border-yellow-400' : 'bg-[#24243e] border border-gray-700'}">
                                <p class="text-xl font-bold mb-2">লেভেল ${v.level}</p>
                                <p class="text-sm text-gray-400">প্রয়োজনীয় রেফারেল: <span class="font-bold">${v.referrals}</span> জন</p>
                                <p class="text-lg font-bold text-blue-300 mt-2">${v.reward}</p>
                            </div>
                        `).join('')}
                    </div>
                    
                    <div class="max-w-md mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl mt-10">
                        <h3 class="text-2xl font-semibold mb-4 text-center">ভিআইপি স্ট্যাটাস চেক করুন</h3>
                        <form id="vip-check-form" onsubmit="handleVIPCheck(event)">
                            <div class="mb-4">
                                <label for="vipEmail" class="block mb-2 text-sm font-medium text-gray-300">আপনার ইমেইল</label>
                                <input type="email" id="vipEmail" name="email" required value="${RM71.user.email || ''}" class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 text-white" placeholder="Email">
                            </div>
                            <div class="mb-6">
                                <label for="vipUid" class="block mb-2 text-sm font-medium text-gray-300">আপনার UID/ID</label>
                                <input type="text" id="vipUid" name="uid" required value="${RM71.user.uid || ''}" class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 text-white" placeholder="UID/ID">
                            </div>
                            <button type="submit" class="w-full bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-4 rounded-lg transition duration-200">
                                স্ট্যাটাস আপডেট/চেক করুন (ডাটাবেস মকড)
                            </button>
                        </form>
                    </div>
                </div>
            `;
        };

        window.renderReportSection = () => `
            <div class="p-6 md:p-10 max-w-xl mx-auto bg-[#24243e] rounded-xl shadow-2xl">
                <h2 class="text-4xl font-bold text-green-400 mb-6 text-center">১৩. রিপোর্ট সেন্টার</h2>
                <form id="report-form" onsubmit="handleReportSubmit(event)">
                    <div class="mb-4">
                        <label for="problem" class="block mb-2 text-sm font-medium text-gray-300">আপনার সমস্যা বিস্তারিত লিখুন</label>
                        <textarea id="problem" name="problem" required rows="4" class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white" placeholder="সমস্যা..."></textarea>
                    </div>
                    <div class="mb-4">
                        <label for="reportUid" class="block mb-2 text-sm font-medium text-gray-300">আপনার UID লিখুন</label>
                        <input type="text" id="reportUid" name="reportUid" required value="${RM71.user.uid || ''}" class="w-full p-3 rounded-lg bg-gray-700 border border-gray-600 focus:ring-green-500 focus:border-green-500 text-white" placeholder="আপনার গেম ID">
                    </div>
                    <div class="mb-6">
                        <label for="screenshot" class="block mb-2 text-sm font-medium text-gray-300">স্ক্রিনশট যোগ করুন (ঐচ্ছিক)</label>
                        <input type="file" id="screenshot" name="screenshot" accept="image/*" class="w-full text-sm text-gray-400 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-green-50 file:text-green-700 hover:file:bg-green-100">
                        <p class="text-xs text-gray-500 mt-1">দ্রষ্টব্য: এই ডেমোতে স্ক্রিনশট আপলোড functionality মকড।</p>
                    </div>
                    <button type="submit" class="w-full bg-red-500 hover:bg-red-600 text-white font-bold py-3 px-4 rounded-lg transition duration-200">
                        রিপোর্ট সাবমিট করুন (ডাটাবেস মকড)
                    </button>
                </form>
            </div>
        `;

        window.renderOwnerSection = () => `
            <div class="p-6 md:p-10">
                <h2 class="text-4xl font-bold text-green-400 mb-6 text-center">১৪ ও ১৫. মালিক এবং টিম সদস্য</h2>
                
                <div class="max-w-2xl mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl mb-8">
                    <h3 class="text-2xl font-semibold mb-4 border-b border-gray-700 pb-2">মালিকের আইডি</h3>
                    <div class="space-y-3">
                        <p class="text-lg text-gray-300">ইনস্টাগ্রাম: <a href="https://instagram.com/__ra__ha__t__" target="_blank" class="text-pink-400 hover:underline">__ra__ha__t__</a></p>
                        <p class="text-lg text-gray-300">টেলিগ্রাম: <a href="tel:+8801828941805" class="text-blue-400 hover:underline">01828941805</a></p>
                    </div>
                </div>

                <div class="max-w-2xl mx-auto bg-[#24243e] p-6 rounded-xl shadow-2xl">
                    <h3 class="text-2xl font-semibold mb-4 border-b border-gray-700 pb-2">RM71 টিম সদস্য</h3>
                    <ul class="list-disc list-inside space-y-3 pl-4">
                        <li class="text-lg text-gray-300">
                            <span class="font-bold text-green-400">১. RAHAT</span>
                            <span class="text-sm text-gray-400"> (মালিক, CO, এবং ডেভেলপার)</span>
                        </li>
                        <li class="text-lg text-gray-300">
                            <span class="font-bold text-green-400">২. MARUR</span>
                            <span class="text-sm text-gray-400"> (অ্যাসিস্ট্যান্ট এবং পার্টনার)</span>
                        </li>
                    </ul>
                </div>
            </div>
        `;

        // --- Helper Components ---
        window.renderFeatureCard = (title, description, page) => `
            <div onclick="navigateTo('${page}')" class="bg-[#24243e] p-6 rounded-xl hover:bg-[#343452] transition duration-300 cursor-pointer shadow-xl border border-gray-700 hover:border-green-500">
                <h3 class="text-2xl font-bold text-white mb-2">${title}</h3>
                <p class="text-gray-400">${description}</p>
                <button class="mt-4 text-green-400 hover:text-green-300 font-semibold flex items-center mx-auto">
                    আরো দেখুন →
                </button>
            </div>
        `;

        window.renderDownloadButton = (title, link, color) => `
            <a href="${link}" target="_blank" class="w-full max-w-sm ${color} hover:bg-opacity-80 text-white font-bold py-4 px-6 rounded-xl transition duration-300 shadow-lg text-xl">
                ${title} ডাউনলোড করুন
            </a>
        `;

        window.renderCodeCard = (title, code, color) => `
            <div class="bg-[#24243e] p-5 rounded-xl border-t-4 ${color} shadow-lg">
                <h3 class="text-xl font-semibold text-white mb-3">${title}</h3>
                <div class="relative">
                    <input type="text" value="${code}" readonly class="w-full p-3 rounded-lg bg-gray-700 text-yellow-300 font-mono text-lg select-all border-dashed border-2 border-gray-600">
                    <button onclick="copyToClipboard('${code}')" class="absolute right-0 top-0 h-full px-4 bg-gray-800 hover:bg-gray-700 rounded-r-lg text-sm text-white transition duration-200">
                        কপি
                    </button>
                </div>
            </div>
        `;

        window.renderEventCard = (title, description, date) => `
            <div class="bg-[#24243e] p-5 rounded-xl shadow-lg border-l-4 border-red-500">
                <h3 class="text-xl font-bold text-red-400 mb-1">${title}</h3>
                <p class="text-sm text-gray-400 mb-3">তারিখ: ${date}</p>
                <p class="text-gray-300">${description}</p>
            </div>
        `;
        
        window.renderStatCard = (title, value, colorClass) => `
            <div class="bg-[#24243e] p-5 rounded-xl shadow-lg border border-gray-700">
                <p class="text-sm text-gray-400 mb-1">${title}</p>
                <p class="text-3xl font-extrabold ${colorClass}">${value}</p>
            </div>
        `;

        window.renderTeamMembers = () => `
            <div class="grid md:grid-cols-2 gap-6">
                <div class="bg-[#24243e] p-6 rounded-xl shadow-xl border-l-4 border-green-500">
                    <h3 class="text-2xl font-bold text-green-400">RAHAT</h3>
                    <p class="text-gray-300">মালিক, CO, এবং ডেভেলপার</p>
                    <p class="text-sm text-gray-400 mt-2">প্রধান লক্ষ্য: নতুন গেমিং ফিচার এবং অ্যাপ স্থায়িত্ব নিশ্চিত করা।</p>
                </div>
                <div class="bg-[#24243e] p-6 rounded-xl shadow-xl border-l-4 border-blue-500">
                    <h3 class="text-2xl font-bold text-blue-400">MARUR</h3>
                    <p class="text-gray-300">অ্যাসিস্ট্যান্ট এবং পার্টনার</p>
                    <p class="text-sm text-gray-400 mt-2">দায়িত্ব: কমিউনিটি ম্যানেজমেন্ট এবং মার্কেটিং সহায়তা।</p>
                </div>
            </div>
        `;

        // --- Utility Functions ---

        window.copyToClipboard = (text) => {
            if (document.execCommand('copy')) {
                const textarea = document.createElement('textarea');
                textarea.value = text;
                document.body.appendChild(textarea);
                textarea.select();
                document.execCommand('copy');
                document.body.removeChild(textarea);
                showMessage('কপি করা হয়েছে!', 'info');
            } else {
                showMessage('কপি করতে সমস্যা হচ্ছে।', 'error');
            }
        };

        window.copyInviteLink = () => {
            const linkInput = document.getElementById('invite-link');
            if (linkInput) {
                copyToClipboard(linkInput.value);
            }
        };
        
        window.selectRedeemPackage = (packageId) => {
            const btn = document.getElementById('redeem-submit-btn');
            const status = document.getElementById('redeem-status');
            const redeemInput = document.getElementById('redeemPackage');
            
            document.querySelectorAll('.redeem-select-btn').forEach(b => {
                b.classList.remove('bg-red-700');
                b.classList.add('bg-yellow-500');
            });
            
            const selectedBtn = event.target;
            selectedBtn.classList.remove('bg-yellow-500');
            selectedBtn.classList.add('bg-red-700');
            
            redeemInput.value = packageId;
            btn.disabled = false;
            btn.classList.remove('bg-gray-500', 'cursor-not-allowed');
            btn.classList.add('bg-green-500', 'hover:bg-green-600');

            const pointsRequired = (packageId === 'redeem1' ? 209 : 709);
            const userPoints = RM71.user.points;

            if (userPoints < pointsRequired) {
                status.textContent = `❌ ${pointsRequired} পয়েন্ট প্রয়োজন। আপনার ব্যালেন্স কম আছে।`;
                status.classList.add('text-red-400');
            } else {
                status.textContent = `✅ ${pointsRequired} পয়েন্ট কেটে নেওয়া হবে। (ডাটাবেস মকড)`;
                status.classList.remove('text-red-400');
                status.classList.add('text-green-400');
            }
        };

        // --- Event Listener Attacher (Run after renderApp) ---
        window.attachEventListeners = () => {
            const buyDiamondForm = document.getElementById('buy-diamond-form');
            if (buyDiamondForm) {
                const packageInput = document.getElementById('package');
                document.querySelectorAll('input[name="package-select-mock"]').forEach(radio => {
                    radio.addEventListener('change', (e) => {
                        packageInput.value = e.target.value;
                    });
                });
            }
        };

        // --- Initialization ---
        document.addEventListener('DOMContentLoaded', () => {
            // সরাসরি অ্যাপ রেন্ডার করা হচ্ছে, Firebase Initialization বাইপাস করা হলো
            renderApp();
        });

    </script>
    
    <!-- কাস্টম মেসেজ বক্স -->
    <div id="message-box"></div>

    <!-- ন্যাভিগেশন বার (NAVBAR) -->
    <nav class="sticky top-0 bg-[#2d2d4d] bg-opacity-95 shadow-lg z-30 p-4 border-b border-gray-700">
        <div class="max-w-7xl mx-auto flex justify-between items-center">
            <div class="text-3xl font-extrabold text-green-400 cursor-pointer" onclick="navigateTo('home')">RM71</div>
            
            <!-- ব্যালেন্স ডিসপ্লে -->
            <div class="flex items-center space-x-4 text-sm md:text-base">
                <span class="text-gray-300 hidden sm:inline">UID: <span id="user-uid-display" class="font-mono text-yellow-300 text-sm">N/A</span></span>
                <div class="flex items-center space-x-1 bg-yellow-900/50 p-1 px-3 rounded-full shadow-inner">
                    <span class="text-yellow-400 font-bold">পয়েন্ট:</span>
                    <span id="points-balance" class="font-extrabold text-white">0</span>
                </div>
                <div class="flex items-center space-x-1 bg-blue-900/50 p-1 px-3 rounded-full shadow-inner">
                    <span class="text-blue-400 font-bold">ডায়মন্ড:</span>
                    <span id="diamond-balance" class="font-extrabold text-white">0</span>
                </div>
            </div>
        </div>
        
        <!-- মোবাইল এবং ডেস্কটপ ন্যাভিগেশন লিংক -->
        <div class="mt-4 overflow-x-auto whitespace-nowrap scrollbar-hide">
            <div class="inline-flex space-x-3 p-1">
                <button onclick="navigateTo('home')" class="nav-btn">হোম</button>
                <button onclick="navigateTo('about')" class="nav-btn">টিম</button>
                <button onclick="navigateTo('downloads')" class="nav-btn">ডাউনলোড</button>
                <button onclick="navigateTo('passkey')" class="nav-btn bg-red-600 hover:bg-red-700">পাসকি জেনারেটর</button>
                <button onclick="navigateTo('giftcode')" class="nav-btn">গিফট কোড</button>
                <button onclick="navigateTo('events')" class="nav-btn">বিশেষ ইভেন্ট</button>
                <button onclick="navigateTo('referral')" class="nav-btn">রেফার ও পয়েন্ট</button>
                <button onclick="navigateTo('buy_diamond')" class="nav-btn">ডায়মন্ড কিনুন</button>
                <button onclick="navigateTo('redeem')" class="nav-btn">পয়েন্ট রিডিম</button>
                <button onclick="navigateTo('vip')" class="nav-btn">ভিআইপি</button>
                <button onclick="navigateTo('report')" class="nav-btn">রিপোর্ট</button>
                <button onclick="navigateTo('owner')" class="nav-btn">মালিক/আইডি</button>
            </div>
        </div>
    </nav>
    
    <!-- প্রধান বিষয়বস্তু কন্টেইনার -->
    <main id="main-content" class="main-container max-w-7xl mx-auto py-8">
        <!-- বিষয়বস্তু JavaScript দ্বারা রেন্ডার করা হবে -->
    </main>

    <!-- ফুটার -->
    <footer class="bg-[#2d2d4d] text-center p-4 text-gray-500 text-sm fixed bottom-0 w-full z-20 border-t border-gray-700">
        <p>&copy; ২০২৫ RM71 Gaming Portal. সমস্ত অধিকার সংরক্ষিত। | App ID: default-app-id</p>
    </footer>

    <!-- CSS ক্লাস যা জাভাস্ক্রিপ্ট দ্বারা ব্যবহার করা হয় -->
    <style>
        .nav-btn {
            @apply px-4 py-2 rounded-lg text-sm font-semibold transition duration-200 bg-gray-700 hover:bg-green-600;
        }
        /* স্ক্রলবার লুকানোর জন্য (মোবাইলের মতো) */
        .scrollbar-hide::-webkit-scrollbar {
            display: none;
        }
        .scrollbar-hide {
            -ms-overflow-style: none;  /* IE and Edge */
            scrollbar-width: none;  /* Firefox */
        }
    </style>
</body>
</html>
