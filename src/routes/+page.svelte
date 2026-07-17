<script lang="ts">
    import { onMount } from 'svelte';
    import type { MediaConnection, Peer as PeerType } from 'peerjs';

    let myPeerId = $state('');
    let joinId: string | null = $state(null);
    let peerIdInput = $state('');
    let status = $state('Connecting to signaling server...');
    let view: 'host' | 'guest' | 'active' = $state('host');
    
    let copyText = $state('Copy Invite Link');
    let isCopied = $state(false);

    let peer: PeerType;
    let localStream: MediaStream | null = null;
    let activeCall: MediaConnection | null = null;
    let remoteAudio: HTMLAudioElement | undefined = $state();

    onMount(async () => {
        const { default: Peer } = await import('peerjs');
        const urlParams = new URLSearchParams(window.location.search);
        joinId = urlParams.get('join');

        let myPersistentId = localStorage.getItem('webrtc_user_id');
        if (!myPersistentId) {
            myPersistentId = crypto.randomUUID();
            localStorage.setItem('webrtc_user_id', myPersistentId);
        }

        peer = new Peer(myPersistentId);

        peer.on('open', (id: string) => {
            myPeerId = id;
            if (joinId) {
                view = 'guest';
                status = "Ready to connect to host.";
            } else {
                status = "Ready. Send the invite link to start a call.";
            }
        });

        peer.on('call', async (call: MediaConnection) => {
            status = "Incoming call... Accessing microphone...";
            try {
                localStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
                call.answer(localStream);
                handleCall(call);
            } catch (err) {
                status = "Microphone access denied.";
                console.error(err);
            }
        });
    });

    async function initiateCall(remoteId: string | null) {
        if (!remoteId) return alert("Please enter a Peer ID");
        status = "Calling... Requesting mic...";
        try {
            localStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
            const call = peer.call(remoteId, localStream);
            handleCall(call);
        } catch (err) {
            status = "Microphone access denied.";
            console.error(err);
        }
    }

    function handleCall(call: MediaConnection) {
        activeCall = call;
        view = 'active';
        status = "Connecting...";

        call.on('stream', (remoteStream: MediaStream) => {
            if (remoteAudio) remoteAudio.srcObject = remoteStream;
            status = "Call Connected!";
        });

        call.on('close', resetUI);
        call.on('error', (err: any) => {
            status = "Call failed.";
            console.error(err);
            resetUI();
        });
    }

    function resetUI() {
        if (localStream) {
            localStream.getTracks().forEach(track => track.stop());
            localStream = null;
        }
        view = joinId ? 'guest' : 'host';
        status = "Call ended.";
    }

    function hangup() {
        if (activeCall) activeCall.close();
        resetUI();
    }

    function decline() {
        window.history.replaceState({}, document.title, window.location.pathname);
        joinId = null;
        view = 'host';
        status = "Ready. Send the invite link to start a call.";
    }

    function copyLink() {
        if (!myPeerId) return;
        const inviteLink = `${window.location.origin}${window.location.pathname}?join=${myPeerId}`;
        
        navigator.clipboard.writeText(inviteLink).then(() => {
            copyText = "Copied! ✓";
            isCopied = true;
            setTimeout(() => {
                copyText = "Copy Invite Link";
                isCopied = false;
            }, 2000);
        }).catch(() => {
            alert("Could not auto-copy. Copy this link manually: " + inviteLink);
        });
    }

    function resetUUID() {
        localStorage.removeItem('webrtc_user_id');
        window.location.reload();
    }
</script>

<div class="flex items-center justify-center min-h-screen bg-slate-100 p-4 font-sans text-slate-800">
    <div class="bg-white p-8 rounded-[2rem] shadow-xl w-full max-w-[400px] text-center">
        <h3 class="mt-0 text-2xl font-extrabold tracking-tight text-slate-900 mb-6">
            WebRTC Audio Call
        </h3>

        {#if view === 'host'}
            <div>
                <div class="bg-slate-50 p-4 rounded-2xl border border-slate-200 mb-6">
                    <div class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-1">Your ID</div>
                    <div class="text-xl font-black text-slate-700 break-all mb-3 tracking-tight">{myPeerId || 'Generating...'}</div>
                    
                    <button 
                        class="w-full py-3 px-4 rounded-xl font-bold transition-all border-2 disabled:opacity-50 {isCopied ? 'bg-emerald-50 text-emerald-600 border-emerald-200' : 'bg-white text-slate-700 border-slate-200 hover:border-slate-300 hover:bg-slate-50'}" 
                        onclick={copyLink} 
                        disabled={!myPeerId}
                    >
                        {copyText}
                    </button>

                    <button 
                        class="w-full mt-2 py-2 px-4 rounded-xl text-xs font-semibold border border-red-200 text-red-600 bg-white hover:bg-red-50/50 transition-all active:scale-[0.98] disabled:opacity-50" 
                        onclick={resetUUID}
                        disabled={!myPeerId}
                    >
                        🔄 Reset Identity (Generate New ID)
                    </button>
                </div>

                <div class="my-6 flex items-center text-xs text-slate-400 font-semibold uppercase tracking-widest">
                    <div class="flex-1 border-t border-slate-200"></div>
                    <span class="px-4">OR</span>
                    <div class="flex-1 border-t border-slate-200"></div>
                </div>

                <input 
                    type="text" 
                    bind:value={peerIdInput} 
                    placeholder="Enter friend's ID" 
                    class="w-full p-3.5 mb-3 bg-white border border-slate-200 rounded-xl text-center text-slate-700 font-medium placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-blue-500"
                >
                <button 
                    class="w-full py-3.5 rounded-xl font-bold transition-all bg-blue-600 text-white hover:bg-blue-700" 
                    onclick={() => initiateCall(peerIdInput)}
                >
                    Start Call
                </button>
            </div>
        {/if}

        {#if view === 'guest'}
            <div>
                <div class="bg-blue-50 p-6 rounded-2xl border border-blue-100 mb-4">
                    <div class="w-12 h-12 bg-blue-100 text-blue-600 rounded-full flex items-center justify-center mx-auto mb-3 text-xl">👋</div>
                    <p class="m-0 mb-4 text-blue-900 text-[0.95rem] font-bold">You have been invited to a call!</p>
                    <button 
                        class="w-full py-3.5 rounded-xl font-bold transition-all bg-blue-600 text-white hover:bg-blue-700" 
                        onclick={() => initiateCall(joinId)}
                    >
                        Join Call Now
                    </button>
                </div>
                <button 
                    class="w-full py-3 rounded-xl font-semibold transition-all bg-white text-slate-500 border border-slate-200 hover:bg-slate-50 hover:text-slate-700" 
                    onclick={decline}
                >
                    Decline
                </button>
            </div>
        {/if}

        {#if view === 'active'}
            <div class="py-4">
                <div class="text-6xl mb-8 animate-bounce">📞</div>
                <button 
                    class="w-full py-3.5 rounded-xl font-bold transition-all bg-red-600 text-white hover:bg-red-700" 
                    onclick={hangup}
                >
                    Hang Up
                </button>
            </div>
        {/if}

        <div class="text-xs text-slate-400 mt-6 min-h-[20px] font-medium tracking-wide">
            {status}
        </div>
        <!-- svelte-ignore a11y_media_has_caption -->
        <audio bind:this={remoteAudio} autoplay></audio>
    </div>
</div>