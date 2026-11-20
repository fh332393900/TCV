# TCV

```sql
SELECT
    'Leasing' AS customerType,
    p.poNumber,
    p.dealerCode,
    d.dealerNameCn AS dealerName,
    IFNULL(p.totalAmount, 0) AS totalServiceFee,
    IFNULL(sf.specialNet, IFNULL(p.specialAmount, 0)) AS adjustSpecialFee,
    IFNULL(p.totalAmount, 0)
      + IFNULL(sf.specialNet, IFNULL(p.specialAmount, 0)) AS actualServiceFee,
    p.paymentStatus,
    sf.type,
    sf.typeName,
    sf.id
FROM ibdsf_po_info_list p
LEFT JOIN ibdsf_dealer_basic_info d
    ON d.dealerCode = p.dealerCode
   AND d.channel = 'BM1'
INNER JOIN (
    SELECT
        s.poNumber,
        s.dealerCode,
        GROUP_CONCAT(DISTINCT s.type ORDER BY s.type) AS type,
        GROUP_CONCAT(
            DISTINCT CASE s.type
                WHEN '1' THEN 'Add'
                WHEN '2' THEN 'Subtract'
                ELSE s.type
            END
            ORDER BY s.type
        ) AS typeName,
        GROUP_CONCAT(DISTINCT s.id ORDER BY s.id) AS id,
        SUM(
            CASE
                WHEN s.type = '2'
                    THEN -ABS(IFNULL(NULLIF(s.specialFeeAmount, 0), IFNULL(s.serviceFee, 0)))
                ELSE ABS(IFNULL(NULLIF(s.specialFeeAmount, 0), IFNULL(s.serviceFee, 0)))
            END
        ) AS specialNet
    FROM ibdsf_bm1_adjust_special_fee s
    WHERE s.status = '1'
      AND (
            s.poNumber = :poMonth
         OR (
                (s.poNumber IS NULL OR s.poNumber = '')
            AND s.month = CAST(:poMonth AS DECIMAL(25,8))
            )
      )
    GROUP BY s.poNumber, s.dealerCode
) sf
    ON sf.dealerCode = p.dealerCode
   AND (sf.poNumber = p.poNumber OR sf.poNumber IS NULL OR sf.poNumber = '')
WHERE p.channel = 'BM1'
  AND p.poNumber = :poMonth
  AND (:dealerCode IS NULL OR :dealerCode = ''
       OR p.dealerCode LIKE CONCAT('%', :dealerCode, '%'))
  AND (:dealerName IS NULL OR :dealerName = ''
       OR d.dealerNameCn LIKE CONCAT('%', :dealerName, '%'))
ORDER BY p.dealerCode
LIMIT :pageIndex, :pageSize;
```
