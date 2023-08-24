<script>
	import { onMount, afterUpdate } from 'svelte';
	import { v4 } from 'uuid';
	import { env } from '$env/dynamic/public';

	let uid = null;
	let socket;
	let transcript;
  let stream;
  let audioDataCache = [];
  let searchQuery = '';
  let recording = false;
  let transcriptDiv;
  let timestamp;

  const scrollToBottom = async (node) => {
		node.scroll({ top: node.scrollHeight, behavior: 'smooth' });
	};

	afterUpdate(() => {
		scrollToBottom(transcriptDiv);
	});

  $: filteredTranscript = transcript ? transcript.segments.filter(segment => segment.text.toLowerCase().includes(searchQuery.toLowerCase())): [];
	
  /**
	 * Resamples the audio data to a target sample rate of 16kHz.
	 * @param {Array|ArrayBuffer|TypedArray} audioData - The input audio data.
	 * @param {number} [origSampleRate=44100] - The original sample rate of the audio data.
	 * @returns {Float32Array} The resampled audio data at 16kHz.
	 */
	function resampleTo16kHZ(audioData, origSampleRate = 44100) {
		// Convert the audio data to a Float32Array
		const data = new Float32Array(audioData);

		// Calculate the desired length of the resampled data
		const targetLength = Math.round(data.length * (16000 / origSampleRate));

		// Create a new Float32Array for the resampled data
		const resampledData = new Float32Array(targetLength);

		// Calculate the spring factor and initialize the first and last values
		const springFactor = (data.length - 1) / (targetLength - 1);
		resampledData[0] = data[0];
		resampledData[targetLength - 1] = data[data.length - 1];

		// Resample the audio data
		for (let i = 1; i < targetLength - 1; i++) {
			const index = i * springFactor;
			const leftIndex = Math.floor(index).toFixed();
			const rightIndex = Math.ceil(index).toFixed();
			const fraction = index - leftIndex;
			resampledData[i] = data[leftIndex] + (data[rightIndex] - data[leftIndex]) * fraction;
		}

		// Return the resampled data
		return resampledData;
	}

	async function startRecording() {
    transcript = null;
		stream = await navigator.mediaDevices.getUserMedia({ audio: true }); // audio stream

		uid = v4(); // generate a unique id for this recording

		if (stream) {
      recording = true;
      timestamp = new Date();
			socket = new WebSocket(env.PUBLIC_WEBSOCKET_URL); // create a websocket connection
			let isServerReady = false;

			socket.onopen = (event) => {
				// when the connection is open send the handshake
				socket.send(
					JSON.stringify({
						uid: uid,
						multilingual: false,
						language: 'en',
						task: 'transcribe'
					})
				);
			};

			socket.onmessage = async (event) => {
				const data = JSON.parse(event.data);

				if (data.uid !== uid) return; // ignore messages that are not for this recording

				if (data?.message && data?.message === 'SERVER_READY') {
					console.log('Server ready');
					isServerReady = true;
					return;
				}

				if (data.message === 'DISCONNECTED') {
					console.log('Server disconnected');
					socket.close();
					return;
				}

        transcript = JSON.parse(event.data);
        transcript.segements = transcript.segments.map(segment => {
          segment.duration = (segment.end - segment.start).toFixed(2);
          segment.start = timestampFromSeconds(segment.start);
          segment.end = timestampFromSeconds(segment.end);
          return segment;
        });

			};

			const context = new AudioContext();
			const mediaStream = context.createMediaStreamSource(stream);
			const recorder = context.createScriptProcessor(4096, 1, 1);
			recorder.onaudioprocess = async (event) => {
				if (!context || !isServerReady) return;

				const inputData = event.inputBuffer.getChannelData(0);
				const audioData16kHz = resampleTo16kHZ(inputData, context.sampleRate);

				audioDataCache.push(inputData);
				socket.send(audioData16kHz);
			};

			mediaStream.connect(recorder);
			recorder.connect(context.destination);


		}
    else {
			if (socket) {
				socket.close();
				audioDataCache = [];
			}
			return;
		}
	}


  function stopRecording() {
    if (stream) {
      // Stop the audio stream tracks
      recording = false;
      const tracks = stream.getTracks();
      tracks.forEach(track => track.stop());

      // Close the WebSocket connection if it's open
      if (socket) {
        socket.close();
        socket = null;
      }

      // Clear any audio data cache or other cleanup actions
      audioDataCache = [];
      
    }
  }

  function recordingButtonClicked() {
    if (recording) {
      stopRecording();
    }
    else {
      startRecording();
    }
  }

  function highlightMatchingWords(text) {
    const query = searchQuery.toLowerCase();
    const highlightedText = text.replace(new RegExp(query, 'gi'), `<span class="highlight">${query}</span>`);
    return highlightedText;
  }


  function downloadTranscript() {
    if (!transcript) return;

    const transcriptText = transcript.segments
      .map((segment, i) => `${i} \n${segment.start} --> ${segment.end} \n${segment.text}`)
      .join('\n');

    const blob = new Blob([transcriptText], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);

    const a = document.createElement('a');
    a.href = url;
    a.download = 'transcript.txt';
    a.click();

    // Clean up by revoking the object URL
    URL.revokeObjectURL(url);
  }

  /**
   * Converts seconds to a timestamp string
   * @param seconds
   */
  function timestampFromSeconds(seconds) {
    const newTimestamp = new Date(timestamp); // Create a new Date object with the same value as the original timestamp
    newTimestamp.setSeconds(newTimestamp.getSeconds() + seconds);
    const formattedTimestamp = newTimestamp.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
    return formattedTimestamp;
  }
  

</script>

<div>
  <div class="nav">
    <button on:click={recordingButtonClicked} class:recording >{recording ? 'Stop transcribing' : 'Start transcribing'}</button>
    <div style="display: flex">
      {#if socket}
        <p style="color: green;">Connected</p>
      {:else}
        <p style="color: red;">Not connected</p>
      {/if}
    </div>
    {#if transcript}
    <div class="download-button">
      <button on:click={downloadTranscript}>Download</button>
    </div>
    {/if}
  </div>

  <input type="text" bind:value={searchQuery} placeholder="Search transcript" />
  <div class="transcript" bind:this={transcriptDiv}>
    {#if filteredTranscript.length}
      {#each filteredTranscript as segment}
        <div class="timestamps">
          <p>Start: {segment.start}</p>
          <p>Duration: {segment.duration}s</p>
          <p>End: {segment.end}</p>
        </div>
        <p>
          {@html highlightMatchingWords(segment.text)}
        </p>
      {/each}
    {:else}
      <p>No matching results</p>
    {/if}
  </div>

</div>

<style>
  div.nav{
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }

  button {
    padding: 0.5rem;
    margin-bottom: 0.5rem;
    font-size: 16px;
    background-color: #282828;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
  }
  /* button:hover {
    background-color: #404040;
    transition: background-color 0.4s ease-in-out;
  } */
  .recording{
    background-color: #fff;
    color: #ff0000;
    border: 1px solid #ff0000;
  }

  input[type="text"] {
    padding: 10px;
    font-size: 16px;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%;
    box-sizing: border-box;
  }
  
	.timestamps {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
	}

	.transcript {
		margin-top: 20px;
    flex: 1;
    overflow: auto;
	}

  :global(.highlight) {
		background-color: rgb(31, 31, 31);
    color: white;
		font-weight: bold;
	}

</style>
