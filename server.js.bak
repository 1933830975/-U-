const express = require('express');
const axios = require('axios');
const path = require('path');
const app = express();
const PORT = process.env.PORT || 3000;

// 中间件
app.use(express.json());
app.use(express.static('public'));

// ========== 工具路由 ==========
// 导航首页
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// USDT 地址管理器（注意路径：public/tools/USDT/index.html）
app.get('/usdt', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'tools', 'USDT', 'index.html'));
});

// 黑客工具包（路径：public/tools/hacker/index.html）
app.get('/hacker', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'tools', 'hacker', 'index.html'));
});

// ========== USDT 工具 API 路由 ==========
const TRC_USDT_CONTRACT = 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t';
const TRONGRID_API = 'https://api.trongrid.io';
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY;
const ERC_USDT_CONTRACT = '0xdAC17F958D2ee523a2206206994597C13D831ec7';

// TRC20 余额查询
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

// TRC20 交易记录
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

// ERC20 余额查询
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

// ERC20 交易记录
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

// 批量查询余额
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

// 查询单个地址的交易记录
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

// 启动服务
app.listen(PORT, () => {
    console.log(`🚀 服务已启动: http://localhost:${PORT}`);
    console.log(`📁 工具导航: http://localhost:${PORT}/`);
    console.log(`💰 USDT 工具: http://localhost:${PORT}/usdt`);
    console.log(`🔧 黑客工具: http://localhost:${PORT}/hacker`);
});