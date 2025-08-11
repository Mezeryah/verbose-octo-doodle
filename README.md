
import cv2
import numpy as np
import json
import hashlib
import hmac
import time
from flask import Flask, request, jsonify

app = Flask(__name__)

# Secret key for signing manifests (in practice, keep this secure)
SECRET_KEY = b'supersecretkey'

def embed_watermark(image: np.ndarray, watermark_text: str) -> np.ndarray:
    """Embed a simple invisible watermark in the image's least significant bit."""
    watermark_bits = ''.join(format(ord(c), '08b') for c in watermark_text)
    flat_image = image.flatten()
    for i, bit in enumerate(watermark_bits):
        flat_image[i] = (flat_image[i] & 0xFE) | int(bit)
    return flat_image.reshape(image.shape)

def extract_watermark(image: np.ndarray, length: int) -> str:
    """Extract the watermark text from the image."""
    flat_image = image.flatten()
    bits = [str(flat_image[i] & 1) for i in range(length * 8)]
    chars = [chr(int(''.join(bits[i:i+8]), 2)) for i in range(0, len(bits), 8)]
    return ''.join(chars)

def create_manifest(prompt_hash, seed, model_version, watermark_id, policy_flags):
    timestamp = int(time.time())
    manifest = {
        'prompt_hash': prompt_hash,
        'seed': seed,
        'model_version': model_version,
        'watermark_id': watermark_id,
        'timestamp': timestamp,
        'policy_flags': policy_flags
    }
    manifest_json = json.dumps(manifest, sort_keys=True)
    signature = hmac.new(SECRET_KEY, manifest_json.encode(), hashlib.sha256).hexdigest()
    manifest['signature'] = signature
    return manifest

@app.route('/verify', methods=['POST'])
def verify():
    data = request.json
    manifest = data.get('manifest')
    # Verify signature
    signature = manifest.pop('signature', None)
    manifest_json = json.dumps(manifest, sort_keys=True)
    expected_sig = hmac.new(SECRET_KEY, manifest_json.encode(), hashlib.sha256).hexdigest()
    synthetic_confidence = 0.98  # Placeholder for actual detection logic

    verification_result = {
        'synthetic_confidence': synthetic_confidence,
        'watermark_id': manifest.get('watermark_id'),
        'model_version': manifest.get('model_version'),
        'manifest_signature_status': (signature == expected_sig)
    }
    return jsonify(verification_result)

if __name__ == '__main__':
    # Load example image
    image = cv2.imread('input.jpg')
    watermark_text = 'wm_12345'
    watermarked_img = embed_watermark(image, watermark_text)
    cv2.imwrite('output_watermarked.png', watermarked_img)

    # Create manifest example
    manifest = create_manifest(
        prompt_hash='abc123',
        seed=42,
        model_version='v1.0',
        watermark_id='wm_12345',
        policy_flags={'consent_verified': True}
    )
    print(json.dumps(manifest, indent=2))

    # Run API
    app.run(port=5000)
