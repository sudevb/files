@Service
public class DeltaSensitivityMapper {

    @Autowired
    private FRateService fRateService;

    public List<SensitivityDto> modelsToDtos(List<Trade> tradeList) {
        return tradeList.parallelStream() // Using parallelStream for better performance
                .flatMap(trade -> processTrade(trade, tradeList).stream())
                .toList();
    }

    private List<SensitivityDto> processTrade(Trade trade, List<Trade> tradeList) {
        LocalDate maturityDate = trade.getMaturityDate().toLocalDate();
        LocalDate asOfDate = trade.getAsOfDate().toLocalDate();

        if (trade.getFxProductInfo() instanceof FXLinear fLinear) {
            Period tradeDuration = Period.between(asOfDate, maturityDate);

            if (!tradeDuration.isNegative()) {
                if (isTradeMtmInvalid(fLinear)) {
                    Pair<FXLinear, Boolean> result = checkPartialTrade(fLinear, tradeList);
                    trade.setFxProductInfo(result.getLeft());
                    if (result.getRight()) {
                        return mapTradeToFxDelta(trade);
                    }
                } else {
                    return mapTradeToFxDelta(trade);
                }
            }
        }
        return List.of();
    }

    private boolean isTradeMtmInvalid(FXLinear fLinear) {
        return Objects.isNull(fLinear.getTradeMtm()) || Objects.equals(fLinear.getTradeMtm(), BigDecimal.ZERO);
    }

    private List<SensitivityDto> mapTradeToFxDelta(Trade trade) {
        List<SensitivityDto> sensitivityDtos = new ArrayList<>();
        if (isEligibleForDeltaAandB(trade)) {
            sensitivityDtos.add(toFxDeltaA(trade));
            sensitivityDtos.add(toFxDeltaB(trade));
        }
        sensitivityDtos.add(toFxDeltaFX(trade));
        return sensitivityDtos;
    }

    private boolean isEligibleForDeltaAandB(Trade trade) {
        String tradeId = trade.getTradeId();
        return !tradeId.contains("T069") && tradeId.contains("MTS");
    }

    private SensitivityDto toFxDeltaA(Trade trade) {
        return createDeltaSensitivity(trade, DeltaType.DELTA_A);
    }

    private SensitivityDto toFxDeltaB(Trade trade) {
        return createDeltaSensitivity(trade, DeltaType.DELTA_B);
    }

    private SensitivityDto toFxDeltaFX(Trade trade) {
        return createDeltaSensitivity(trade, DeltaType.DELTA_FX);
    }

    private SensitivityDto createDeltaSensitivity(Trade trade, DeltaType deltaType) {
        FXLinear fLinear = (FXLinear) trade.getFxProductInfo();
        LocalDate maturityDate = trade.getMaturityDate().toLocalDate();
        LocalDate asOfDate = trade.getAsOfDate().toLocalDate();

        SensitivityDto dto = new SensitivityDto();
        dto.setAsOfDate(asOfDate.toString());
        dto.setTradeId(trade.getTradeId());
        dto.setSensitivityName(deltaType.getName());
        dto.setRiskClass(deltaType.getRiskClass());

        // Shared logic for DV01 calculation
        double dv01 = calculateDv01(fLinear, asOfDate, maturityDate, deltaType);
        dto.setValues(List.of(BigDecimal.valueOf(dv01)));

        // Risk factor setup
        String riskFactor = getRiskFactor(fLinear, deltaType);
        dto.setRiskFactor(riskFactor);

        dto.setTenorLabels(List.of(maturityDate.toString()));
        dto.setTenorDates(List.of(maturityDate.toString()));
        dto.setValueCurrency(getValueCurrency(fLinear, deltaType));

        return dto;
    }

    private double calculateDv01(FXLinear fLinear, LocalDate asOfDate, LocalDate maturityDate, DeltaType deltaType) {
        double usdDf = Math.abs(fLinear.getTradeMtm().doubleValue() / (getFvNonUsd(fLinear) - fLinear.getBuyNotional().doubleValue()));
        long numDays = asOfDate.until(maturityDate, ChronoUnit.DAYS);
        double timeFrac = (double) numDays / 365.0;
        return -timeFrac * fLinear.getBuyNotional().doubleValue() * usdDf / 10000.0;
    }

    private double getFvNonUsd(FXLinear fLinear) {
        return fLinear.getSellNotional().doubleValue() / fLinear.getRevalRate();
    }

    private String getRiskFactor(FXLinear fLinear, DeltaType deltaType) {
        return switch (deltaType) {
            case DELTA_A -> fLinear.getBuyCurrency() + "_CURVE_A";
            case DELTA_B -> fLinear.getSellCurrency() + "_CURVE_B";
            case DELTA_FX -> fLinear.getSellCurrency() + "_CURVE_FX";
        };
    }

    private String getValueCurrency(FXLinear fLinear, DeltaType deltaType) {
        return deltaType == DeltaType.DELTA_FX ? fLinear.getSellCurrency() : fLinear.getBuyCurrency();
    }

    private Pair<FXLinear, Boolean> checkPartialTrade(FXLinear fLinear, List<Trade> tradeList) {
        String[] parts = fLinear.getTradeId().split("_");
        if (parts.length < 2) {
            return Pair.of(fLinear, Boolean.FALSE);
        }
        String tradeId = parts[1];
        return tradeList.stream()
                .filter(trade -> trade.getTradeId().equalsIgnoreCase(tradeId))
                .findFirst()
                .map(matchedTrade -> {
                    if (matchedTrade.getFxProductInfo() instanceof FXLinear matchedFxLinear) {
                        fLinear.setTradeMtm(matchedFxLinear.getTradeMtm());
                        return Pair.of(fLinear, Boolean.TRUE);
                    }
                    return Pair.of(fLinear, Boolean.FALSE);
                })
                .orElse(Pair.of(fLinear, Boolean.FALSE));
    }

    // Enum for delta types to differentiate specific behavior
    private enum DeltaType {
        DELTA_A("Delta A", "Interest Rate"),
        DELTA_B("Delta B", "Interest Rate"),
        DELTA_FX("Delta FX", "FX");

        private final String name;
        private final String riskClass;

        DeltaType(String name, String riskClass) {
            this.name = name;
            this.riskClass = riskClass;
        }

        public String getName() {
            return name;
        }

        public String getRiskClass() {
            return riskClass;
        }
    }
}
