WITH CurrentPartition AS (
  SELECT maxValueDate AS actualValueDate 
  FROM `db-uat-xs6i-mp-palace-acc.revenue_data_service.TB_REPORT_COB`
),

BaseData AS (
  SELECT
    rds.accountingData.accountType AS AccountType,
    DATE(rds.vd) AS AsOfDate,
    rds.transactionData.amount AS Amount,
    EXTRACT(YEAR FROM rds.vd) AS YearNo,
    EXTRACT(MONTH FROM rds.vd) AS MonthNo,
    EXTRACT(QUARTER FROM rds.vd) AS QuarterNo,
    rds.le AS LegalEntity,
    rds.*
  FROM `db-uat-xs6i-mp-palace-acc.revenue_data_service.RDS_BALANCES` rds
  JOIN (
    SELECT currentPackageld, valueDate, legalEntity
    FROM `db-uat-xs6i-mp-palace-acc.revenue_data_service.RDS_EXECUTION_PACKAGES`
    WHERE palaceBalId IN (
      SELECT palaceBalId 
      FROM `db-uat-xs6i-mp-palace-acc.revenue_data_service.RDS_EXECUTION_PACKAGES`
      WHERE palaceBalId LIKE 'PALACE-BAL%'
      QUALIFY ROW_NUMBER() OVER (
        PARTITION BY valueDate, legalEntity, postingDate 
        ORDER BY CAST(REGEXP_EXTRACT(palaceBalId, r'Z_(\d+)$') AS INT64) DESC
      ) = 1
    )
    AND usedForDbp = TRUE
  ) derived
  ON rds.package = derived.currentPackageld
  AND rds.vd = derived.valueDate
  AND rds.le = derived.legalEntity
  WHERE rds.vd >= DATE_SUB((SELECT actualValueDate FROM CurrentPartition), INTERVAL 64 DAY)
),

Calc AS (
  SELECT
    *,
    LAG(Amount) OVER (
      PARTITION BY AccountType, LegalEntity, EXTRACT(YEAR FROM AsOfDate)
      ORDER BY AsOfDate
    ) AS PrevAmount
  FROM BaseData
)

SELECT
  CAST(package AS STRING) AS PackageId,
  identifierData_feedid AS FeedId,
  transactionData_measurementType AS AccountingMeasurementType,
  transactionData_transactionalCurrencyCode AS TransactionalCurrencyCode,
  DATE(vd) AS AsOfDate,
  transactionData_periodType AS PeriodType,
  transactionData_dealid AS DealId,
  transactionData_debitCreditIndicator AS SourceDebitCreditIndicator,
  transactionData_amount AS Amount,
  transactionData_dealDomain AS DealDomain,
  accountingData_accountType AS AccountType,

  -- 🟡 MTD calculation
  CASE 
    WHEN AccountType = 'PnL' THEN
      CASE 
        WHEN EXTRACT(MONTH FROM AsOfDate) = 1 THEN Amount
        ELSE Amount - COALESCE(PrevAmount, 0)
      END
    WHEN AccountType = 'BS' THEN Amount - COALESCE(PrevAmount, 0)
  END AS MTD,

  -- 🟣 QTD calculation
  CASE 
    WHEN AccountType = 'PnL' THEN Amount
    WHEN AccountType = 'BS' THEN
      SUM(
        Amount - COALESCE(PrevAmount, 0)
      ) OVER (
        PARTITION BY AccountType, LegalEntity, YearNo, QuarterNo
        ORDER BY AsOfDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
      )
  END AS QTD,

  -- ... keep all your other columns below ...
  enrichmentData_counterparty_role AS ClientParticipantRole,
  enrichmentData_counterparty_domain AS CounterpartyDomain,
  enrichmentData_counterparty_id AS CounterpartyId,
  enrichmentData_dbParticipant_role AS AccountingDBParticipantCptyRole,
  le AS LegalEntityId

FROM Calc;
