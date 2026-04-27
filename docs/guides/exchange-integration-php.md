# Exchange Integration — PHP

This guide is a PHP-specific companion to the [Exchange Integration Guide](exchange-integration.md). All five examples below run against the public mainnet endpoint `https://rpc.solenchain.io` (chain ID `1`) and the explorer at `https://api.solenchain.io`.

## Prerequisites

- **PHP 7.2+** — needed for libsodium (`sodium_*` functions) which provides Ed25519.
- **GMP extension** — for safe big-integer math: `apt install php-gmp` (Debian/Ubuntu) or `pecl install gmp`.
- **BLAKE3** — not built into PHP. Pick one:
    - **PECL extension** (recommended for production throughput): `pecl install blake3` then `extension=blake3` in `php.ini`.
    - **Pure-PHP package** via Composer (slower but zero ops setup): `composer require cyphrme/blake3` (or any PHP package that exposes a BLAKE3 hash function).
    - **Shell out to `b3sum`** (only for prototyping; one process spawn per signed tx is too slow for production).

The code below assumes a `blake3(string $bytes): string` function returning **raw binary** (32 bytes). Wire it to whichever option you chose — see [`blake3()` helper](#blake3-helper) at the bottom.

For the canonical byte-level signing spec, see [Transaction Signing](../specs/transaction-signing.md). The pinned cross-language test vectors at [`tools/vectors/vectors.json`](https://github.com/Solen-Blockchain/solen/blob/main/tools/vectors/vectors.json) let you sanity-check your PHP signer against the Rust reference before sending real funds.

---

## Shared Client

All five examples use a small RPC + Base58 helper. Save as `Solen.php`:

```php
<?php
declare(strict_types=1);

class SolenClient
{
    public function __construct(
        private string $rpcUrl = 'https://rpc.solenchain.io',
        private string $apiUrl = 'https://api.solenchain.io',
        private int    $chainId = 1, // 1 = mainnet, 9000 = testnet, 1337 = devnet
        private int    $timeoutSeconds = 10,
    ) {}

    /** JSON-RPC 2.0 call. Returns the decoded `result` field or throws on error. */
    public function rpc(string $method, array $params = []): mixed
    {
        $body = json_encode([
            'jsonrpc' => '2.0',
            'id'      => 1,
            'method'  => $method,
            'params'  => $params,
        ], JSON_UNESCAPED_SLASHES);

        $ch = curl_init($this->rpcUrl);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => $body,
            CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
            CURLOPT_TIMEOUT        => $this->timeoutSeconds,
        ]);
        $resp = curl_exec($ch);
        if ($resp === false) {
            $err = curl_error($ch);
            curl_close($ch);
            throw new RuntimeException("RPC transport error: $err");
        }
        curl_close($ch);

        $decoded = json_decode($resp, true);
        if (!is_array($decoded)) {
            throw new RuntimeException("RPC: invalid JSON response: $resp");
        }
        if (isset($decoded['error'])) {
            throw new RuntimeException(
                "RPC error {$decoded['error']['code']}: {$decoded['error']['message']}"
            );
        }
        return $decoded['result'] ?? null;
    }

    /** GET against the explorer REST API. */
    public function explorer(string $path): mixed
    {
        $ch = curl_init($this->apiUrl . $path);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => $this->timeoutSeconds,
        ]);
        $resp = curl_exec($ch);
        $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        if ($resp === false || $code >= 400) {
            throw new RuntimeException("Explorer GET $path failed (HTTP $code)");
        }
        return json_decode($resp, true);
    }

    public function chainId(): int { return $this->chainId; }
}

/** Bitcoin-alphabet Base58 encoder (no checksum — Solen does NOT use Base58Check). */
function base58_encode(string $bytes): string
{
    if ($bytes === '') return '';
    $alphabet = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';

    $zeros = 0;
    while ($zeros < strlen($bytes) && $bytes[$zeros] === "\x00") $zeros++;

    $num = gmp_init(bin2hex($bytes), 16);
    $out = '';
    while (gmp_cmp($num, 0) > 0) {
        [$num, $rem] = [gmp_div_q($num, 58), gmp_intval(gmp_mod($num, 58))];
        $out = $alphabet[$rem] . $out;
    }
    return str_repeat('1', $zeros) . $out;
}

/** Decodes Base58 to raw bytes. Throws on invalid characters. */
function base58_decode(string $s): string
{
    if ($s === '') return '';
    $alphabet = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';
    $table = array_flip(str_split($alphabet));

    $zeros = 0;
    while ($zeros < strlen($s) && $s[$zeros] === '1') $zeros++;

    $num = gmp_init(0);
    for ($i = 0; $i < strlen($s); $i++) {
        if (!isset($table[$s[$i]])) {
            throw new InvalidArgumentException("invalid base58 char: {$s[$i]}");
        }
        $num = gmp_add(gmp_mul($num, 58), $table[$s[$i]]);
    }
    $hex = gmp_strval($num, 16);
    if (strlen($hex) % 2 === 1) $hex = '0' . $hex;
    return str_repeat("\x00", $zeros) . hex2bin($hex);
}

/** Decodes either Base58 or 64-char hex (with optional 0x prefix) to 32 raw bytes. */
function parse_address(string $s): string
{
    $s = trim($s);
    $clean = (str_starts_with($s, '0x')) ? substr($s, 2) : $s;
    if (strlen($clean) === 64 && ctype_xdigit($clean)) {
        return hex2bin($clean);
    }
    $bytes = base58_decode($s);
    if (strlen($bytes) !== 32) {
        throw new InvalidArgumentException("address must decode to 32 bytes");
    }
    return $bytes;
}
```

---

## 1. Send SOLEN

```php
<?php
require_once 'Solen.php';

/**
 * Sign and submit a Transfer.
 *
 * @param string $senderSeedHex  64-char hex of the 32-byte Ed25519 seed (private key)
 * @param string $recipient      Base58 or hex address
 * @param int    $amount         Base units (1 SOLEN = 100_000_000 base units)
 * @param int    $maxFee         Upper bound on fee debited (base units)
 */
function sendTransfer(
    SolenClient $client,
    string $senderSeedHex,
    string $recipient,
    int    $amount,
    int    $maxFee = 100_000,
): array {
    $seed = hex2bin($senderSeedHex);
    if (strlen($seed) !== 32) throw new InvalidArgumentException("seed must be 32 bytes");

    // Derive Ed25519 keypair. libsodium uses a 64-byte secret key (seed || pubkey).
    $kp     = sodium_crypto_sign_seed_keypair($seed);
    $secret = sodium_crypto_sign_secretkey($kp);
    $sender = sodium_crypto_sign_publickey($kp);    // 32 bytes — this IS the address

    $recipientBytes = parse_address($recipient);

    // 1. Get next nonce (accounts for pending mempool txs).
    $senderB58 = base58_encode($sender);
    $nonce = (int) $client->rpc('solen_getNextNonce', [$senderB58]);

    // 2. Build the action. Field order is consensus-critical; serde reads
    //    Rust struct declaration order: Transfer { to, amount }.
    $action = [
        'Transfer' => [
            'to'     => array_values(unpack('C*', $recipientBytes)),  // 32-element JSON array of u8
            'amount' => $amount,
        ],
    ];
    $actions = [$action];

    // 3. Canonicalize actions to JSON. PHP's json_encode preserves insertion
    //    order for assoc arrays, which matches Rust serde's struct order. Slashes
    //    must NOT be escaped (Rust does not escape them).
    $actionsJson = json_encode($actions, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);
    $actionsHash = blake3($actionsJson); // 32 raw bytes

    // 4. Build the 96-byte signing message.
    //    Layout: chain_id[8 LE] ‖ sender[32] ‖ nonce[8 LE] ‖ max_fee[16 LE] ‖ actions_hash[32]
    $msg  = pack('P', $client->chainId());     // u64 LE
    $msg .= $sender;                           // 32 bytes
    $msg .= pack('P', $nonce);                 // u64 LE
    $msg .= pack('P', $maxFee) . str_repeat("\x00", 8); // u128 LE (low 8 bytes + 8 zero bytes)
    $msg .= $actionsHash;                      // 32 bytes
    assert(strlen($msg) === 96);

    // 5. Sign with Ed25519.
    $signature = sodium_crypto_sign_detached($msg, $secret); // 64 bytes

    // 6. Submit.
    $result = $client->rpc('solen_submitOperation', [[
        'sender'    => array_values(unpack('C*', $sender)),
        'nonce'     => $nonce,
        'actions'   => $actions,
        'max_fee'   => $maxFee,
        'signature' => array_values(unpack('C*', $signature)),
    ]]);

    return $result; // { accepted: bool, error: string|null }
}

// Example
$client = new SolenClient();
$result = sendTransfer(
    $client,
    senderSeedHex: 'YOUR_64_CHAR_HEX_SEED',
    recipient:     'dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur',
    amount:        150_000_000_000, // 1500 SOLEN
);
print_r($result);
```

!!! warning "Amount above 2^63"
    PHP's `int` is signed 64-bit. SOLEN's total supply (2 × 10^17 base units) fits comfortably, but if you ever need to sign for `amount` ≥ 2^63 you'll need a custom JSON serializer that emits the value as an unquoted decimal literal — `json_encode` cannot serialize a string-as-number without quotes. Use GMP for the math and `str_replace` the placeholder string into the encoded JSON.

!!! warning "max_fee above 2^63"
    Same caveat as `amount`. The `pack('P', …)` line above writes only the low 8 bytes of the `u128`; for fees above `2^63` you need to write the full 16 bytes (e.g. via GMP-derived hex split into two `pack('P', …)` halves).

---

## 2. Get Balance

```php
<?php
require_once 'Solen.php';

function getBalance(SolenClient $client, string $address): string
{
    // Returns a string to preserve precision (balance is u128).
    return (string) $client->rpc('solen_getBalance', [$address]);
}

$client = new SolenClient();
$bal = getBalance($client, 'dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur');
echo "Balance: $bal base units (" . bcdiv($bal, '100000000', 8) . " SOLEN)\n";
```

For full account info (nonce, code hash, auth methods) use `solen_getAccount` instead:

```php
$account = $client->rpc('solen_getAccount', [$address]);
// $account = ['id' => '...', 'balance' => '...', 'nonce' => 42, 'code_hash' => '...', ...]
```

---

## 3. Node Status

```php
<?php
require_once 'Solen.php';

function getNodeStatus(SolenClient $client): array
{
    return $client->rpc('solen_chainStatus');
}

$client = new SolenClient();
$status = getNodeStatus($client);

echo "Height:           {$status['height']}\n";
echo "Pending ops:      {$status['pending_ops']}\n";
echo "Total staked:     {$status['total_staked']}\n";
echo "Total circulation:{$status['total_circulation']}\n";
echo "Block time (ms):  {$status['config']['block_time_ms']}\n";
```

**Health check pattern.** Poll once a minute and alert if `height` hasn't advanced — a healthy node bumps height every ~6 seconds.

```php
function isNodeHealthy(SolenClient $client, int &$lastHeight, int &$lastChange): bool
{
    $h = (int) $client->rpc('solen_chainStatus')['height'];
    $now = time();
    if ($h > $lastHeight) {
        $lastHeight = $h;
        $lastChange = $now;
    }
    // Alert if no progress for 60 seconds (10× normal block time).
    return ($now - $lastChange) < 60;
}
```

---

## 4. Transaction Status

Solen identifies a transaction by `(block_height, tx_index)`. The explorer REST API is the simplest way to look one up:

```php
<?php
require_once 'Solen.php';

function getTransaction(SolenClient $client, int $blockHeight, int $txIndex): array
{
    return $client->explorer("/api/tx/$blockHeight/$txIndex");
}

$client = new SolenClient();
$tx = getTransaction($client, blockHeight: 5000, txIndex: 0);

echo "Sender:  {$tx['sender']}\n";
echo "Nonce:   {$tx['nonce']}\n";
echo "Success: " . ($tx['success'] ? 'true' : 'false') . "\n";
echo "Gas:     {$tx['gas_used']}\n";
if (!$tx['success']) echo "Error:   {$tx['error']}\n";
```

**Wait-for-confirmation pattern** (when you only know the sender + nonce — e.g. you just submitted and want to block until it lands):

```php
function waitForConfirmation(
    SolenClient $client,
    string $sender,
    int $nonce,
    int $timeoutSeconds = 30,
): ?array {
    $deadline = time() + $timeoutSeconds;
    while (time() < $deadline) {
        $account = $client->rpc('solen_getAccount', [$sender]);
        // If on-chain nonce moved past our nonce, our tx (or one with the same
        // nonce slot) is included. Look it up via the account tx history.
        if ((int)$account['nonce'] > $nonce) {
            $txs = $client->explorer("/api/accounts/$sender/txs?limit=20");
            foreach ($txs as $t) {
                if ((int)$t['nonce'] === $nonce) return $t;
            }
        }
        sleep(2);
    }
    return null; // Timed out — could still land later.
}
```

Solen has **deterministic finality** — once a tx is in a block, it is irreversible. One confirmation is enough. There are no reorgs.

---

## 5. Transaction History (Polling)

For exchanges that can't subscribe to events (or want a backstop against missed websockets), the canonical pattern is to scan finalized blocks for transfer events matching your deposit-address set. This scales to millions of addresses because the per-event lookup is `O(1)`.

```php
<?php
require_once 'Solen.php';

class DepositScanner
{
    /** @var array<string, true>  Set of Base58 addresses you watch. */
    private array $depositAddresses;

    /** Last height already processed — persist this to your DB. */
    private int $lastProcessedHeight;

    public function __construct(
        private SolenClient $client,
        array $depositAddresses,
        int $startHeight,
    ) {
        $this->depositAddresses    = array_fill_keys($depositAddresses, true);
        $this->lastProcessedHeight = $startHeight;
    }

    /** Call once per block-time interval (~6s). Returns deposits found. */
    public function scan(): array
    {
        $current = (int) $this->client->rpc('solen_chainStatus')['height'];
        $deposits = [];

        for ($h = $this->lastProcessedHeight + 1; $h <= $current; $h++) {
            $txs = $this->client->explorer("/api/blocks/$h/txs") ?? [];
            foreach ($txs as $tx) {
                if (!$tx['success']) continue;
                foreach (($tx['events'] ?? []) as $ev) {
                    if (($ev['topic'] ?? '') !== 'transfer') continue;
                    $data = $ev['data'] ?? '';
                    if (strlen($data) < 96) continue;

                    // Event data layout: recipient[32] ‖ amount[16, LE u128]
                    $recipientHex = substr($data, 0, 64);
                    $amountHex    = substr($data, 64, 32);

                    $recipientB58 = base58_encode(hex2bin($recipientHex));
                    if (!isset($this->depositAddresses[$recipientB58])) continue;

                    $deposits[] = [
                        'block_height' => $h,
                        'tx_index'     => (int)$tx['index'],
                        'sender'       => $tx['sender'],
                        'recipient'    => $recipientB58,
                        'amount'       => self::parseLeU128($amountHex),
                    ];
                }
            }
            $this->lastProcessedHeight = $h;
            // PERSIST $this->lastProcessedHeight to your DB here so a crash
            // doesn't replay or skip blocks on restart.
        }
        return $deposits;
    }

    /** Parse a 32-char hex string as little-endian u128, returning a decimal string. */
    private static function parseLeU128(string $hex): string
    {
        // Reverse byte order to get big-endian, then parse as GMP big int.
        $beHex = '';
        for ($i = strlen($hex) - 2; $i >= 0; $i -= 2) {
            $beHex .= substr($hex, $i, 2);
        }
        return gmp_strval(gmp_init($beHex, 16), 10);
    }
}

// Usage
$client = new SolenClient();
$scanner = new DepositScanner(
    $client,
    depositAddresses: [
        'dJNVRKH1Jh2TYBrjJUfNwQrB5b1S8xiCW926yEKWiur',
        'H9we5VhjE42uCXXgy1YTcFV5bZReBcqxznMNkryHz8KY',
        // ...load from DB
    ],
    startHeight: getLastProcessedFromDb() ?? 0,
);

while (true) {
    foreach ($scanner->scan() as $deposit) {
        // credit the user, write to DB, etc.
        echo "Deposit {$deposit['amount']} → {$deposit['recipient']} at block {$deposit['block_height']}\n";
    }
    sleep(6); // one block interval
}
```

**Per-address polling** (acceptable only for small address sets, e.g. <100):

```php
function getAccountHistory(SolenClient $client, string $address, int $limit = 50): array
{
    return $client->explorer("/api/accounts/$address/txs?limit=$limit");
}
```

---

## 6. Generate a Keypair

Solen uses Ed25519 keypairs. The **account address IS the public key**, Base58-encoded. libsodium is already a prerequisite of this guide, so no extra extension is needed.

```php
<?php
require_once 'Solen.php';

// Generate a cryptographically secure 32-byte seed (the private key).
$seed = random_bytes(32);

// Derive the Ed25519 keypair from the seed. The public key IS the address.
$kp        = sodium_crypto_sign_seed_keypair($seed);
$publicKey = sodium_crypto_sign_publickey($kp); // 32 bytes

$address      = base58_encode($publicKey);
$privateKey   = bin2hex($seed);       // 64-char hex — feed this to sendTransfer()
$publicKeyHex = bin2hex($publicKey);

echo "Address:     $address\n";
echo "Private Key: $privateKey\n";
echo "Public Key:  $publicKeyHex\n";
```

The `$privateKey` hex string is exactly what `sendTransfer()` in [§1](#1-send-solen) expects as `senderSeedHex`.

!!! warning "Key Storage"
    The private key (seed) must be stored securely — it is the only way to sign transactions from this address, and there is no recovery if it is lost or leaked. **Always use `random_bytes()`**, never `rand()` / `mt_rand()` / `uniqid()` — those are not cryptographically secure and will produce guessable keys. Today the seed directly generates the keypair (no derivation path); a BIP-39 / SLIP-0010 path under SLIP-0044 coin type `20260424` is planned once the [SLIP registration](https://github.com/satoshilabs/slips/pull/2010) merges. See [Transaction Signing → HD Derivation](../specs/transaction-signing.md#hd-derivation).

---

## `blake3()` Helper

Wire ONE of these to your install — the rest of the code calls `blake3($bytes): string` expecting **raw 32 bytes** out:

=== "PECL extension"
    ```php
    function blake3(string $data): string {
        // The pecl/blake3 extension exposes blake3() globally; the second arg
        // selects raw binary output.
        return blake3_hash($data, true);
    }
    ```

=== "Composer (cyphrme/blake3)"
    ```php
    use Cyphrme\Blake3\Blake3;
    function blake3(string $data): string {
        return Blake3::hash($data); // returns 32 raw bytes
    }
    ```

=== "Shell out to b3sum (prototyping only)"
    ```php
    function blake3(string $data): string {
        $proc = proc_open(['b3sum', '--raw'],
            [0 => ['pipe','r'], 1 => ['pipe','w']], $pipes);
        fwrite($pipes[0], $data);
        fclose($pipes[0]);
        $hash = stream_get_contents($pipes[1]);
        fclose($pipes[1]);
        proc_close($proc);
        return $hash;
    }
    ```

### Verifying your signer against the official test vectors

Before sending real funds, prove your PHP signer matches the Rust reference by reproducing one entry from [`tools/vectors/vectors.json`](https://github.com/Solen-Blockchain/solen/blob/main/tools/vectors/vectors.json):

```php
$seed = hex2bin('1111111111111111111111111111111111111111111111111111111111111111');
$kp   = sodium_crypto_sign_seed_keypair($seed);
$pub  = sodium_crypto_sign_publickey($kp);
$sec  = sodium_crypto_sign_secretkey($kp);

// Take the canonical actions JSON straight from the vectors file:
$actionsJson = '[{"Transfer":{"to":[23,203,121,251,43,65,32,242,177,236,101,228,25,141,110,8,178,142,129,63,235,1,228,164,0,131,155,133,225,128,128,206],"amount":1500000000}}]';

$msg  = pack('P', 1);                                    // chain_id = 1
$msg .= $pub;
$msg .= pack('P', 0);                                    // nonce = 0
$msg .= pack('P', 100000) . str_repeat("\x00", 8);       // max_fee = 100_000 (low u64 + zero high)
$msg .= blake3($actionsJson);

$sig = sodium_crypto_sign_detached($msg, $sec);
echo bin2hex($sig) . "\n";
// Expected:
//   a09196808274e828ff4a24ed551147d28ba211057bfe2af7b616e90ab989ab93
//   66fb990228b648ca1e9a67de600c15012d628bf7d86d85b787cc4f65f7ccb20e
```

If the signature matches the vectors-file `signature_hex`, your signer is correctly wired. If not, the most common culprits are:

1. JSON serialization differs (whitespace, slash-escaping, byte-array form, field order).
2. BLAKE3 implementation produces wrong bytes — verify with `b3sum` against the same input.
3. `pack('P', ...)` not behaving as little-endian on an unusual architecture (extremely rare).

---

## Support

For integration help: [discord.com/invite/QGxv8bebJc](https://discord.com/invite/QGxv8bebJc) · [t.me/solenblockchain](https://t.me/solenblockchain).
