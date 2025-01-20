DECLARE @dateID DATE = '2024-09-27';

-- Calculate JPY FX rate
DECLARE @jpy_fx FLOAT;
SELECT @jpy_fx = (FX.bid + FX.ask) / 2
FROM DFSBatch.MKT_FX FX
WHERE FX.dateID = @dateID AND FX.currencyID = 'JPY';

-- Consolidated query
SELECT 
    R.psTradeId AS TradeId,
    'FX01' AS product,
    CASE R.underlying
        WHEN 'USD/JPY-SPOT' THEN 'JPY'
        ELSE LEFT(R.underlying, 3)
    END AS bucket,
    CASE R.currencyID
        WHEN 'CNY' THEN 'CNH'
        ELSE R.currencyID
    END AS gridCurID,
    CASE R.currencyID
        WHEN 'CNY' THEN 'CNH'
        ELSE R.currencyID
    END AS currencyID,
    'FX01' + CASE R.underlying
        WHEN 'USD/JPY-SPOT' THEN 'JPY'
        ELSE LEFT(R.underlying, 3)
    END AS point,
    SUM(
        CASE 
            WHEN R.underlying LIKE '_/JPY-SPOT' THEN 
                R.delta * @jpy_fx * 
                CASE FX.direction 
                    WHEN 'Y' THEN 1 
                    ELSE 0 
                END * (1 - CASE FX.direction WHEN 'N' THEN 4 ELSE 0 END * SQUARE(1 / (2 * FX.rate)))
            ELSE 
                R.delta * 
                CASE FX.direction 
                    WHEN 'Y' THEN FX.rate 
                    ELSE 1 / FX.rate 
                END
        END
    ) AS delta_frn
FROM dfsoptionsensitivity R
LEFT JOIN fxrate FX 
    ON FX.dateID = @dateID AND FX.currencyID = 
       CASE R.underlying
            WHEN 'USD/JPY-SPOT' THEN 'JPY'
            ELSE LEFT(R.underlying, 3)
       END
WHERE R.asOfDate = @dateID
GROUP BY 
    R.psTradeId,
    R.underlying,
    R.currencyID;
