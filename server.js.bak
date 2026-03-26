const express = require('express');
const fs = require('fs').promises;
const path = require('path');
const axios = require('axios');
const app = express();
const PORT = process.env.PORT || 3000;

const DATA_FILE = path.join(__dirname, 'addresses.json');

app.use(express.json());
app.use(express.static('public')); // 前端静态文件放在 public 目录
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// 读取地址列表（文件存储，重启会丢失）
async function readAddresses() {
    try {
        const data = await fs.readFile(DATA_FILE, 'utf8');
        return JSON.parse(data);
    } catch (err) {
        return [];
    }
}

async function writeAddresses(addresses) {
    await fs.writeFile(DATA_FILE, JSON.stringify(addresses, null, 2));
}

// TRC20 配置
const TRC_USDT_CONTRACT = 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t';
const TRONGRID_API = 'https://api.trongrid.io';

// ERC20 配置
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY;
const ERC_USDT_CONTRACT = '0xdAC17F958D2ee523a2206206994597C13D831ec7';

if (!ETHERSCAN_API_KEY) {
    console.warn('⚠️ 警告: 未设置 ETHERSCAN_API_KEY，ERC20 查询将失败');
}

// 查询 TRC20 地址的余额和 TRX
async function getTrcBalance(address) {
    try {
        const url = `${TRONGRID_API}/v1/accounts/${address}`;
        const resp = await axios.get(url);
        if (!resp.data.data || resp.data.data.length === 0) {
            return { usdtBalance: '0.000000', trxBalance: '0.000000' };
        }
        const account = resp.data.data[0];
        const trxBalance = (account.balance / 1e6).toFixed(6);

        let usdtBalance = 0;
        const trc20Tokens = account.trc20 || [];
        for (const token of trc20Tokens) {
            if (token[TRC_USDT_CONTRACT]) {
                usdtBalance = token[TRC_USDT_CONTRACT] / 1e6;
                break;
            }
        }
        return { usdtBalance: usdtBalance.toFixed(6), trxBalance };
    } catch (err) {
        console.error(`TRC20 查询失败 ${address}:`, err.message);
        return { usdtBalance: '0.000000', trxBalance: '0.000000', error: err.message };
    }
}

// 查询 TRC20 交易记录（最近30笔）
async function getTrcTransactions(address, limit = 30) {
    try {
        const url = `${TRONGRID_API}/v1/accounts/${address}/transactions/trc20?limit=${limit}&contract_address=${TRC_USDT_CONTRACT}&only_confirmed=true`;
        const resp = await axios.get(url);
        const txs = resp.data.data || [];
        return txs.map(tx => {
            const from = tx.from;
            const to = tx.to;
            const value = (parseInt(tx.value) / 1e6).toFixed(6);
            const direction = from === address ? 'OUT' : 'IN';
            const timestamp = new Date(parseInt(tx.block_timestamp)).toLocaleString();
            return { hash: tx.transaction_id, from, to, value, direction, timestamp };
        });
    } catch (err) {
        console.error(`TRC20 交易查询失败 ${address}:`, err.message);
        return [];
    }
}

// 查询 ERC20 余额
async function getErcBalance(address) {
    if (!ETHERSCAN_API_KEY) {
        return { usdtBalance: '0.000000', error: 'Etherscan API Key 未配置' };
    }
    try {
        const url = `https://api.etherscan.io/api?module=account&action=tokenbalance&contractaddress=${ERC_USDT_CONTRACT}&address=${address}&tag=latest&apikey=${ETHERSCAN_API_KEY}`;
        const resp = await axios.get(url);
        if (resp.data.status === '1') {
            const raw = resp.data.result;
            const usdtBalance = (raw / 1e6).toFixed(6);
            return { usdtBalance };
        } else {
            return { usdtBalance: '0.000000', error: resp.data.message };
        }
    } catch (err) {
        console.error(`ERC20 余额查询失败 ${address}:`, err.message);
        return { usdtBalance: '0.000000', error: err.message };
    }
}

// 查询 ERC20 交易记录
async function getErcTransactions(address, limit = 30) {
    if (!ETHERSCAN_API_KEY) return [];
    try {
        const url = `https://api.etherscan.io/api?module=account&action=tokentx&contractaddress=${ERC_USDT_CONTRACT}&address=${address}&sort=desc&apikey=${ETHERSCAN_API_KEY}`;
        const resp = await axios.get(url);
        if (resp.data.status !== '1') return [];
        const txs = resp.data.result.slice(0, limit);
        return txs.map(tx => {
            const from = tx.from;
            const to = tx.to;
            const value = (parseInt(tx.value) / 1e6).toFixed(6);
            const direction = from.toLowerCase() === address.toLowerCase() ? 'OUT' : 'IN';
            const timestamp = new Date(parseInt(tx.timeStamp) * 1000).toLocaleString();
            return { hash: tx.hash, from, to, value, direction, timestamp };
        });
    } catch (err) {
        console.error(`ERC20 交易查询失败 ${address}:`, err.message);
        return [];
    }
}

// ---------- API 路由 ----------
app.get('/api/addresses', async (req, res) => {
    try {
        const addresses = await readAddresses();
        res.json({ addresses });
    } catch (err) {
        res.status(500).json({ error: '读取地址列表失败' });
    }
});

app.post('/api/addresses', async (req, res) => {
    const { address } = req.body;
    if (!address) return res.status(400).json({ error: '地址不能为空' });
    if (!(address.startsWith('0x') || (address.startsWith('T') && address.length === 34))) {
        return res.status(400).json({ error: '地址格式无效' });
    }
    try {
        const addresses = await readAddresses();
        if (addresses.includes(address)) {
            return res.status(400).json({ error: '地址已存在' });
        }
        addresses.push(address);
        await writeAddresses(addresses);
        res.json({ success: true });
    } catch (err) {
        res.status(500).json({ error: '添加失败' });
    }
});

app.delete('/api/addresses/:address', async (req, res) => {
    const address = decodeURIComponent(req.params.address);
    try {
        let addresses = await readAddresses();
        if (!addresses.includes(address)) {
            return res.status(404).json({ error: '地址不存在' });
        }
        addresses = addresses.filter(a => a !== address);
        await writeAddresses(addresses);
        res.json({ success: true });
    } catch (err) {
        res.status(500).json({ error: '删除失败' });
    }
});

app.post('/api/balances', async (req, res) => {
    const { addresses } = req.body;
    if (!addresses || !Array.isArray(addresses)) {
        return res.status(400).json({ error: '需要提供地址数组' });
    }

    const results = [];
    for (const addr of addresses) {
        const type = addr.startsWith('0x') ? 'erc' : (addr.startsWith('T') && addr.length === 34 ? 'trc' : 'unknown');
        if (type === 'trc') {
            const { usdtBalance, trxBalance, error } = await getTrcBalance(addr);
            results.push({ address: addr, type: 'trc', usdtBalance, trxBalance, error });
        } else if (type === 'erc') {
            const { usdtBalance, error } = await getErcBalance(addr);
            results.push({ address: addr, type: 'erc', usdtBalance, error });
        } else {
            results.push({ address: addr, type: 'unknown', usdtBalance: '0.000000', error: '不支持的地址类型' });
        }
    }
    res.json(results);
});

app.get('/api/transactions', async (req, res) => {
    const { address, limit = 30 } = req.query;
    if (!address) return res.status(400).json({ error: '地址不能为空' });
    const type = address.startsWith('0x') ? 'erc' : (address.startsWith('T') && address.length === 34 ? 'trc' : 'unknown');
    if (type === 'unknown') return res.status(400).json({ error: '不支持的地址类型' });

    try {
        let transactions = [];
        if (type === 'trc') {
            transactions = await getTrcTransactions(address, parseInt(limit));
        } else {
            transactions = await getErcTransactions(address, parseInt(limit));
        }
        res.json({ success: true, transactions });
    } catch (err) {
        res.status(500).json({ error: '查询交易记录失败: ' + err.message });
    }
});

app.listen(PORT, () => {
    console.log(`🚀 服务已启动: http://localhost:${PORT}`);
    console.log(`📁 地址数据存储: ${DATA_FILE}`);
});