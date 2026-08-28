# wyze-secure-stream
Secure Wyze API integration with end-to-end encryption for live feed streaming
Wyze Camera
     │
     │ Wyze's authenticated/encrypted stream
     ▼
Your Python gateway
     │
     │ 1. Authenticate device
     │ 2. Establish encrypted session
     │ 3. Authorize camera
     │
     │ AES-GCM encrypted application stream
     ▼
Your authorized phone/device
# Python 3.11+
#
# pip install cryptography

import os
import base64

from cryptography.hazmat.primitives.asymmetric import x25519, ed25519
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.ciphers.aead import AESGCM


class DeviceSession:
    """
    Creates an application-level encrypted session between
    your gateway and one authorized device.
    """

    def __init__(self):
        # Ephemeral X25519 key used for this session.
        self.private_key = x25519.X25519PrivateKey.generate()
        self.public_key = self.private_key.public_key()

        self.aes_key = None

    def public_key_bytes(self):
        return self.public_key.public_bytes_raw()

    def establish(self, client_public_key_bytes: bytes):
        """
        Perform X25519 key agreement and derive an AES-256 key.
        """

        client_public = x25519.X25519PublicKey.from_public_bytes(
            client_public_key_bytes
        )

        shared_secret = self.private_key.exchange(client_public)

        self.aes_key = HKDF(
            algorithm=hashes.SHA256(),
            length=32,
            salt=None,
            info=b"my-wyze-camera-stream-v1",
        ).derive(shared_secret)

    def encrypt(self, plaintext: bytes) -> bytes:
        if self.aes_key is None:
            raise RuntimeError("Session has not been established")

        # Never reuse a nonce with the same AES-GCM key.
        nonce = os.urandom(12)

        ciphertext = AESGCM(self.aes_key).encrypt(
            nonce,
            plaintext,
            b"wyze-live-stream-v1",
        )

        return nonce + ciphertext

    def decrypt(self, encrypted: bytes) -> bytes:
        if self.aes_key is None:
            raise RuntimeError("Session has not been established")

        nonce = encrypted[:12]
        ciphertext = encrypted[12:]

        return AESGCM(self.aes_key).decrypt(
            nonce,
            ciphertext,
            b"wyze-live-stream-v1",
        )


class DeviceIdentity:
    """
    Long-lived identity belonging to ONE authorized device.

    The private key should live in secure device storage,
    not in your Python server.
    """

    def __init__(self):
        self.private_key = ed25519.Ed25519PrivateKey.generate()
        self.public_key = self.private_key.public_key()

    def sign(self, message: bytes) -> bytes:
        return self.private_key.sign(message)

    def public_key_bytes(self):
        return self.public_key.public_bytes_raw()
Device registration
       │
       ▼
Generate Ed25519 identity
       │
       ▼
Register public key with server
       │
       ▼
Server associates:
    device_id
       +
    public_key
       +
    allowed cameras
       │
       ▼
Future connection:
    server → random challenge
    device → signature(challenge)
    server → verifies signature
       │
       ▼
Create ephemeral X25519 session
       │
       ▼
AES-256-GCM encrypted stream
Camera
  │
  │ LAN only
  ▼
Private streaming gateway
  │
  │ authenticated + encrypted
  ▼
Your device