@Service
public class SensitivityService {
    @Autowired
    private DfsOptionSensitivityRepository sensitivityRepo;
    @Autowired
    private FxRateRepository fxRateRepo;

    public List<ProcessedData> calculateDeltas(LocalDate asOfDate) {
        List<DfsOptionSensitivity> sensitivities = sensitivityRepo.findByAsOfDate(asOfDate);

        return sensitivities.stream().map(sensitivity -> {
            String baseCcy = "USD";
            String counterCcy = "JPY";

            // Determine base_ccy and counter_ccy based on the underlying
            if (sensitivity.getUnderlying().equals("USD/JPY-SPOT")) {
                baseCcy = "USD";
                counterCcy = "JPY";
            } else if (List.of("GBP", "EUR", "NZD", "CLF").contains(sensitivity.getCurrency())) {
                baseCcy = sensitivity.getCurrency();
                counterCcy = "USD";
            } else {
                counterCcy = sensitivity.getCurrency();
            }

            // Fetch FX rate
            FxRate fxRate = fxRateRepo.findByBaseCcyAndCounterCcyAndAsOfDate(baseCcy, counterCcy, asOfDate)
                                      .orElseThrow(() -> new RuntimeException("FX rate not found"));

            double fxDirection = fxRate.getDirection().equals("Y") ? 1 : 0;

            // Apply the formula
            double rateComponent = fxDirection - ((1 - fxDirection) * 4 * Math.pow(1 / (2 * fxRate.getRate()), 2));
            double adjustedDelta = sensitivity.getDelta() * fxRate.getRate() * rateComponent;

            // Return processed data
            return new ProcessedData(
                sensitivity.getPsTradeId(),
                sensitivity.getCurrency(),
                sensitivity.getUnderlying(),
                adjustedDelta
            );
        }).collect(Collectors.toList());
    }
}
