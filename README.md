@Service
public class SensitivityService {

    @Autowired
    private DfsOptionSensitivityRepository sensitivityRepo;

    @Autowired
    private FxRateRepository fxRateRepo;

    @Autowired
    private DfsPlValueRepository plValueRepo;

    public List<TradeDelta> calculateDeltas(LocalDate asOfDate) {
        List<TradeDelta> result = new ArrayList<>();

        // Fetch sensitivity data
        List<DfsOptionSensitivity> sensitivities = sensitivityRepo.findByAsOfDate(asOfDate);

        // Step 1: Process `rtor` equivalent
        Map<String, BigDecimal> bucketDeltas = new HashMap<>();
        for (DfsOptionSensitivity sensitivity : sensitivities) {
            String bucket = sensitivity.getUnderlying().equals("USD/JPY-SPOT") ? "JPY" : sensitivity.getUnderlying().substring(0, 3);
            bucketDeltas.merge(bucket, sensitivity.getDelta(), BigDecimal::add);
        }

        // Step 2: Calculate delta for buckets excluding JPY
        for (Map.Entry<String, BigDecimal> entry : bucketDeltas.entrySet()) {
            String bucket = entry.getKey();
            BigDecimal delta = entry.getValue();

            if (!"JPY".equals(bucket)) {
                // Handle special currencies for base_ccy and counter_ccy
                String baseCcy = isSpecialCurrency(bucket) ? bucket : "USD";
                String counterCcy = isSpecialCurrency(bucket) ? "USD" : bucket;

                FxRate fxRate = fxRateRepo.findRate(asOfDate, baseCcy, counterCcy)
                        .orElseThrow(() -> new RuntimeException("FX rate not found for bucket: " + bucket));
                BigDecimal rate = fxRate.getRate();
                BigDecimal directionFactor = "N".equals(fxRate.getDirection()) ? BigDecimal.ONE : BigDecimal.ZERO;

                BigDecimal adjustedDelta = delta.multiply(rate).multiply(
                        directionFactor.subtract(
                                BigDecimal.ONE.subtract(directionFactor)
                                        .multiply(BigDecimal.ONE.divide(rate, 10, RoundingMode.HALF_UP).pow(2))
                        )
                );
                result.add(new TradeDelta(null, bucket, adjustedDelta));
            }
        }

        // Step 3: Process for JPY
        FxRate jpyRate = fxRateRepo.findRate(asOfDate, "USD", "JPY")
                .orElseThrow(() -> new RuntimeException("FX rate not found for JPY"));
        BigDecimal jpyRateValue = jpyRate.getRate();
        BigDecimal jpyDirectionFactor = "N".equals(jpyRate.getDirection()) ? BigDecimal.ONE : BigDecimal.ZERO;

        BigDecimal jpyDelta = bucketDeltas.getOrDefault("JPY", BigDecimal.ZERO).multiply(
                jpyDirectionFactor.add(
                        BigDecimal.ONE.subtract(jpyDirectionFactor)
                                .divide(jpyRateValue, 10, RoundingMode.HALF_UP)
                )
        );

        result.add(new TradeDelta(null, "JPY", jpyDelta));

        // Step 4: Process present values from `dfsplvalue`
        List<DfsPlValue> plValues = plValueRepo.findByAsOfDateAndCurrencyNot(asOfDate, "USD");
        for (DfsPlValue plValue : plValues) {
            // Handle special currencies for base_ccy and counter_ccy
            String baseCcy = isSpecialCurrency(plValue.getCurrency()) ? plValue.getCurrency() : "USD";
            String counterCcy = isSpecialCurrency(plValue.getCurrency()) ? "USD" : plValue.getCurrency();

            FxRate fxRate = fxRateRepo.findRate(asOfDate, baseCcy, counterCcy)
                    .orElseThrow(() -> new RuntimeException("FX rate not found for currency: " + plValue.getCurrency()));

            BigDecimal rate = fxRate.getRate();
            BigDecimal directionFactor = "N".equals(fxRate.getDirection()) ? BigDecimal.ONE : BigDecimal.ZERO;

            BigDecimal adjustedDelta = plValue.getPresentValue().multiply(
                    directionFactor.equals(BigDecimal.ZERO) ? BigDecimal.ONE.negate()
                            .multiply(BigDecimal.ONE.divide(rate, 10, RoundingMode.HALF_UP).pow(2))
                            : BigDecimal.ONE
            );

            result.add(new TradeDelta(plValue.getPsTradeId(), plValue.getCurrency(), adjustedDelta));
        }

        return result;
    }

    // Helper method to check if a currency is a special currency
    private boolean isSpecialCurrency(String currency) {
        return List.of("AUD", "EUR", "GBP", "CLF", "NZD").contains(currency);
    }
}
