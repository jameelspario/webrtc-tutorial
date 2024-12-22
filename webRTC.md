
## What Is WebRTC?

The WebRTC protocol (stands for Web Real-Time Communication), which allows for real-time communication, such as audio & video streaming and generic data sent between multiple clients and establishing direct peer-to-peer connections.


### The Signaling Server

The signaling server is responsible for resolving and establishing a connection between peers to allow peers to connect with each other by exposing minimized private information

### Session Description Protocol

a standard format for describing multimedia communication sessions for a peer-to-peer connection.

The SDP includes some information about the peer connection, such as Codec, source address, media types of audio and video, and other associated properties

```
o=- 5168502270746789779 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 35 36 123 122 125
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:7WRY
a=ice-pwd:XHR3zWN/W59V38h1C6z3s6q+
a=ice-options:trickle renomination
a=fingerprint:sha-256 D7:8A:18:28:84:8D:F7:28:ED:21:99:24:CE:05:93:43:EB:32:1A:E7:6B:72:AB:75:EF:42:46:0E:DD:84:E2:82
a=setup:active
a=mid:0
..
```

Peers exchanged SDP messages

WebRTC uses ICE (Interactive Connectivity Establishment) protocol, and peers can negotiate the actual connection between them by exchanging ICE candidates

### WebRTC on Android

#### Gradle Setup

```
// Add the dependency below to your app level `build.gradle` file:
dependencies {
  implementation "io.getstream:stream-webrtc-android:1.1.1"
}
```

#### PeerConnection

```Kotlin
class StreamPeerConnection(
  private val coroutineScope: CoroutineScope,
  private val type: StreamPeerType,
  private val mediaConstraints: MediaConstraints,
  private val onStreamAdded: ((MediaStream) -> Unit)?,
  private val onNegotiationNeeded: ((StreamPeerConnection, StreamPeerType) -> Unit)?,
  private val onIceCandidate: ((IceCandidate, StreamPeerType) -> Unit)?,
  private val onVideoTrack: ((RtpTransceiver?) -> Unit)?
) : PeerConnection.Observer {

  suspend fun createOffer(): Result<SessionDescription> {
    logger.d { "[createOffer] #sfu; #$typeTag; no args" }
    return createValue { connection.createOffer(it, mediaConstraints) }
  }

  suspend fun createAnswer(): Result<SessionDescription> {
    logger.d { "[createAnswer] #sfu; #$typeTag; no args" }
    return createValue { connection.createAnswer(it, mediaConstraints) }
  }

  suspend fun setRemoteDescription(sessionDescription: SessionDescription): Result<Unit> {
    logger.d { "[setRemoteDescription] #sfu; #$typeTag; answerSdp: ${sessionDescription.stringify()}" }
    return setValue {
      connection.setRemoteDescription(
        it,
        SessionDescription(
          sessionDescription.type,
          sessionDescription.description.mungeCodecs()
        )
      )
    }.also {
      pendingIceMutex.withLock {
        pendingIceCandidates.forEach { iceCandidate ->
          logger.i { "[setRemoteDescription] #sfu; #subscriber; pendingRtcIceCandidate: $iceCandidate" }
          connection.addRtcIceCandidate(iceCandidate)
        }
        pendingIceCandidates.clear()
      }
    }
  }

  suspend fun setLocalDescription(sessionDescription: SessionDescription): Result<Unit> {
    val sdp = SessionDescription(
      sessionDescription.type,
      sessionDescription.description.mungeCodecs()
    )
    logger.d { "[setLocalDescription] #sfu; #$typeTag; offerSdp: ${sessionDescription.stringify()}" }
    return setValue { connection.setLocalDescription(it, sdp) }
  }

  suspend fun addIceCandidate(iceCandidate: IceCandidate): Result<Unit> {
    if (connection.remoteDescription == null) {
      logger.w { "[addIceCandidate] #sfu; #$typeTag; postponed (no remoteDescription): $iceCandidate" }
      pendingIceMutex.withLock {
        pendingIceCandidates.add(iceCandidate)
      }
      return Result.failure(RuntimeException("RemoteDescription is not set"))
    }
    logger.d { "[addIceCandidate] #sfu; #$typeTag; rtcIceCandidate: $iceCandidate" }
    return connection.addRtcIceCandidate(iceCandidate).also {
      logger.v { "[addIceCandidate] #sfu; #$typeTag; completed: $it" }
    }
  }

  ..
}
```

#### PeerConnectionFactory

PeerConnection.Factory is responsible for creating PeerConnection, and it helps you to create video/audio tracks/sources, provide internal loggers, and establish peer connections

```kotlin
class StreamPeerConnectionFactory constructor(
  private val context: Context
) {

  // rtcConfig contains STUN and TURN servers list
  val rtcConfig = PeerConnection.RTCConfiguration(
    arrayListOf(
      // adding google's standard server
      PeerConnection.IceServer.builder("stun:stun.l.google.com:19302").createIceServer()
    )
  ).apply {
    // it's very important to use new unified sdp semantics PLAN_B is deprecated
    sdpSemantics = PeerConnection.SdpSemantics.UNIFIED_PLAN
  }

  ..
 }
 ```

 The StreamPeerConnectionFactory initializes the PeerConnectionFactory and sets up video/audio encoder and decoder, error callbacks, and internal loggers for improving reusability

 ```kotlin
 private val factory by lazy {
  PeerConnectionFactory.initialize(
    PeerConnectionFactory.InitializationOptions.builder(context).createInitializationOptions()
  )

  PeerConnectionFactory.builder()
    .setVideoDecoderFactory(videoDecoderFactory)
    .setVideoEncoderFactory(videoEncoderFactory)
    .setAudioDeviceModule(
      JavaAudioDeviceModule
        .builder(context)
        .setUseHardwareAcousticEchoCanceler(Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q)
        .setUseHardwareNoiseSuppressor(Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q)
        .setAudioRecordErrorCallback(..)
        .setAudioTrackErrorCallback(..)
        .setAudioRecordStateCallback(..)
        .setAudioTrackStateCallback(..)
        .createAudioDeviceModule().also {
          it.setMicrophoneMute(false)
          it.setSpeakerMute(false)
        }
    )
    .createPeerConnectionFactory()
}
```

#### Signaling Client

#### WebRtcSessionManager

we need to create a real connection and observe/handle the signaling responses.

```kotlin
class WebRtcSessionManagerImpl(
  private val context: Context,
  override val signalingClient: SignalingClient,
  override val peerConnectionFactory: StreamPeerConnectionFactory
) : WebRtcSessionManager {
  private val logger by taggedLogger("Call:LocalWebRtcSessionManager")

  private val sessionManagerScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

  // used to send remote video track to the sender
  private val _remoteVideoSinkFlow = MutableSharedFlow<VideoTrack>()
  override val remoteVideoSinkFlow: SharedFlow<VideoTrack> = _remoteVideoSinkFlow

  // declaring video constraints and setting OfferToReceiveVideo to true
  // this step is mandatory to create valid offer and answer
  private val mediaConstraints = MediaConstraints().apply {
    mandatory.addAll(
      listOf(
        MediaConstraints.KeyValuePair("OfferToReceiveAudio", "true"),
        MediaConstraints.KeyValuePair("OfferToReceiveVideo", "true")
      )
    )
  }

  private val peerConnection: StreamPeerConnection by lazy {
    peerConnectionFactory.makePeerConnection(
      coroutineScope = sessionManagerScope,
      configuration = peerConnectionFactory.rtcConfig,
      type = StreamPeerType.SUBSCRIBER,
      mediaConstraints = mediaConstraints,
      onIceCandidateRequest = { iceCandidate, _ ->
        signalingClient.sendCommand(
          SignalingCommand.ICE,
          "${iceCandidate.sdpMid}$ICE_SEPARATOR${iceCandidate.sdpMLineIndex}$ICE_SEPARATOR${iceCandidate.sdp}"
        )
      },
      onVideoTrack = { rtpTransceiver ->
        val track = rtpTransceiver?.receiver?.track() ?: return@makePeerConnection
        if (track.kind() == MediaStreamTrack.VIDEO_TRACK_KIND) {
          val videoTrack = track as VideoTrack
          sessionManagerScope.launch {
            _remoteVideoSinkFlow.emit(videoTrack)
          }
        }
      }
    )
  }

 ..
}
```

Initializing the peer connection defines actions for sending an ICE candidate and handling ream-time Transceiver. RtpTransceiver represents a combination of a real-time sender and receiver, and it provides MediaStreamTrack that renders video/audio with multiple VideoSink.

need to initialize the video-relevant components to display your camera rendering screen on your local device and send the video data to peers in real-time

```kotlin
// getting front camera
private val videoCapturer: VideoCapturer by lazy { buildCameraCapturer() }

// we need it to initialize video capturer
private val surfaceTextureHelper = SurfaceTextureHelper.create(
  "SurfaceTextureHelperThread",
  peerConnectionFactory.eglBaseContext
)

private val videoSource by lazy {
  peerConnectionFactory.makeVideoSource(videoCapturer.isScreencast).apply {
    videoCapturer.initialize(surfaceTextureHelper, context, this.capturerObserver)
    videoCapturer.startCapture(480, 720, 30)
  }
}

private val localVideoTrack: VideoTrack by lazy {
  peerConnectionFactory.makeVideoTrack(
    source = videoSource,
    trackId = "Video${UUID.randomUUID()}"
  )
}

private fun buildCameraCapturer(): VideoCapturer {
  val manager = cameraManager ?: throw RuntimeException("CameraManager was not initialized!")

  val ids = manager.cameraIdList
  var foundCamera = false
  var cameraId = ""

  for (id in ids) {
    val characteristics = manager.getCameraCharacteristics(id)
    val cameraLensFacing = characteristics.get(CameraCharacteristics.LENS_FACING)

    if (cameraLensFacing == CameraMetadata.LENS_FACING_FRONT) {
      foundCamera = true
      cameraId = id
    }
  }

  if (!foundCamera && ids.isNotEmpty()) {
    cameraId = ids.first()
  }

  val camera2Capturer = Camera2Capturer(context, cameraId, null)
  return camera2Capturer
}
```

- VideoCapturer: This is an interface for abstracting the basic abstract behaviors of CameraCapturer. We can start to capture using mobile device’s cameras (facing front or back) with specific resolutions.

- VideoSource: VideoSource is used to create video tracks and add VideoProcessor, which is a lightweight abstraction for an object that can receive video frames, process them, and pass them on to another object.

- VideoTrack: VideoTrack is one of the essential concepts in WebRTC for Android. It manages multiple VideoSink objects, which receive a stream of video frames in real-time and it allows you to control the VideoSink objects, such as adding, removing, enabling, and disabling.

need to observe the real-time signaling responses from the signaling server and handle the SDP offer/answer and ICE candidates with the peerConnection instance

```kotlin
init {
  sessionManagerScope.launch {
    signalingClient.signalingCommandFlow
      .collect { commandToValue ->
        when (commandToValue.first) {
          SignalingCommand.OFFER -> handleOffer(commandToValue.second)
          SignalingCommand.ANSWER -> handleAnswer(commandToValue.second)
          SignalingCommand.ICE -> handleIce(commandToValue.second)
          else -> Unit
        }
      }
  }
}

private fun handleOffer(sdp: String) {
  logger.d { "[SDP] handle offer: $sdp" }
  offer = sdp
}

private suspend fun handleAnswer(sdp: String) {
  logger.d { "[SDP] handle answer: $sdp" }
  peerConnection.setRemoteDescription(
    SessionDescription(SessionDescription.Type.ANSWER, sdp)
  )
}

private suspend fun handleIce(iceMessage: String) {
  val iceArray = iceMessage.split(ICE_SEPARATOR)
  peerConnection.addIceCandidate(
    IceCandidate(
      iceArray[0],
      iceArray[1].toInt(),
      iceArray[2]
    )
  )
}
```

create an SDP offer and answer that should be sent to the signaling server for exchanging media information between peers.

```kotlin
override fun onSessionScreenReady() {
  peerConnection.connection.addTrack(localVideoTrack)
  peerConnection.connection.addTrack(localAudioTrack)
  sessionManagerScope.launch {
    // sending local video track to show local video from start
    _localVideoSinkFlow.emit(localVideoTrack)

    if (offer != null) {
      sendAnswer()
    } else {
      sendOffer()
    }
  }
}

private suspend fun sendOffer() {
  val offer = peerConnection.createOffer().getOrThrow()
  val result = peerConnection.setLocalDescription(offer)
  result.onSuccess {
    signalingClient.sendCommand(SignalingCommand.OFFER, offer.description)
  }
  logger.d { "[SDP] send offer: ${offer.stringify()}" }
}

private suspend fun sendAnswer() {
  peerConnection.setRemoteDescription(
    SessionDescription(SessionDescription.Type.OFFER, offer)
  )
  val answer = peerConnection.createAnswer().getOrThrow()
  val result = peerConnection.setLocalDescription(answer)
  result.onSuccess {
    signalingClient.sendCommand(SignalingCommand.ANSWER, answer.description)
  }
  logger.d { "[SDP] send answer: ${answer.stringify()}" }
}
```

