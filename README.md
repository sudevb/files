@Data
@AllArgsConstructor
@NoArgsConstructor
public class RtorDto {
    private String tradeId;
    private String bucket;
    private String currency;
    private BigDecimal delta;
}




@Service
public class SensitivityService {

    @Autowired
    private DfsOptionSensitivityRepository sensitivityRepo;

    @Autowired
    private FxRateRepository fxRateRepo;

    @Autowired
    private DfsPlValueRepository plValueRepo;

    public List<TradeDelta> calculateDeltas(LocalDate asOfDate) {
        // Step 1: Create the RTOR equivalent
        List<RtorDto> rtorList = createRtor(asOfDate);

        // Step 2: Calculate the final delta values
        List<TradeDelta> result = calculateFinalDeltas(rtorList, asOfDate);

        return result;
    }

    private List<RtorDto> createRtor(LocalDate asOfDate) {
        List<DfsOptionSensitivity> sensitivities = sensitivityRepo.findByAsOfDateAndUnderlyingPattern(asOfDate, "__/JPY-SPOT");

        Map<String, RtorDto> rtorMap = new HashMap<>();
        for (DfsOptionSensitivity sensitivity : sensitivities) {
            String bucket = sensitivity.getUnderlying().equals("USD/JPY-SPOT") ? "JPY" : sensitivity.getUnderlying().substring(0, 3);
            String key = sensitivity.getPsTradeId() + "|" + bucket + "|" + sensitivity.getCurrency();

            rtorMap.compute(key, (k, v) -> {
                if (v == null) {
                    return new RtorDto(sensitivity.getPsTradeId(), bucket, sensitivity.getCurrency(), sensitivity.getDelta());
                }
                v.setDelta(v.getDelta().add(sensitivity.getDelta()));
                return v;
            });
        }

        return new ArrayList<>(rtorMap.values());
    }

    private List<TradeDelta> calculateFinalDeltas(List<RtorDto> rtorList, LocalDate asOfDate) {
        List<TradeDelta> result = new ArrayList<>();

        for (RtorDto rtor : rtorList) {
            if (!"JPY".equals(rtor.getBucket())) {
                // Handle special currencies
                String baseCcy = isSpecialCurrency(rtor.getBucket()) ? rtor.getBucket() : "USD";
                String counterCcy = isSpecialCurrency(rtor.getBucket()) ? "USD" : rtor.getBucket();

                FxRate fxRate = fxRateRepo.findRate(asOfDate, baseCcy, counterCcy)
                        .orElseThrow(() -> new RuntimeException("FX rate not found for bucket: " + rtor.getBucket()));

                BigDecimal rate = fxRate.getRate();
                BigDecimal directionFactor = "N".equals(fxRate.getDirection()) ? BigDecimal.ONE : BigDecimal.ZERO;

                BigDecimal adjustedDelta = rtor.getDelta().multiply(rate).multiply(
                        directionFactor.subtract(
                                BigDecimal.ONE.subtract(directionFactor)
                                        .multiply(BigDecimal.ONE.divide(rate, 10, RoundingMode.HALF_UP).pow(2))
                        )
                );
                result.add(new TradeDelta(rtor.getTradeId(), rtor.getBucket(), adjustedDelta));
            }
        }

        // Process JPY bucket
        FxRate jpyRate = fxRateRepo.findRate(asOfDate, "USD", "JPY")
                .orElseThrow(() -> new RuntimeException("FX rate not found for JPY"));

        BigDecimal jpyRateValue = jpyRate.getRate();
        BigDecimal jpyDirectionFactor = "N".equals(jpyRate.getDirection()) ? BigDecimal.ONE : BigDecimal.ZERO;

        for (RtorDto rtor : rtorList) {
            if ("JPY".equals(rtor.getBucket())) {
                BigDecimal jpyDelta = rtor.getDelta().multiply(
                        jpyDirectionFactor.add(
                                BigDecimal.ONE.subtract(jpyDirectionFactor)
                                        .divide(jpyRateValue, 10, RoundingMode.HALF_UP)
                        )
                );
                result.add(new TradeDelta(rtor.getTradeId(), "JPY", jpyDelta));
            }
        }

        return result;
    }

    private boolean isSpecialCurrency(String currency) {
        return List.of("AUD", "EUR", "GBP", "CLF", "NZD").contains(currency);
    }
}







@Repository
public interface DfsOptionSensitivityRepository extends JpaRepository<DfsOptionSensitivity, Long> {
    @Query("SELECT r FROM DfsOptionSensitivity r WHERE r.asOfDate = :asOfDate AND r.underlying LIKE :underlyingPattern")
    List<DfsOptionSensitivity> findByAsOfDateAndUnderlyingPattern(@Param("asOfDate") LocalDate asOfDate, @Param("underlyingPattern") String underlyingPattern);
}
