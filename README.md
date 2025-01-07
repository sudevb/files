import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;

public class MapAggregator {

    public static void main(String[] args) {
        // Example input maps
        Map<String, Map<String, BigDecimal>> t69Map = new HashMap<>();
        Map<String, Map<String, BigDecimal>> mtsMap = new HashMap<>();

        // Populate maps with example data
        t69Map.put("USD", new HashMap<>(Map.of(
                "PROP", BigDecimal.valueOf(100),
                "CMGG", BigDecimal.valueOf(200)
        )));
        t69Map.put("JPY", new HashMap<>(Map.of(
                "PROP", BigDecimal.valueOf(150),
                "CMGG", BigDecimal.valueOf(250)
        )));

        mtsMap.put("USD", new HashMap<>(Map.of(
                "MTD", BigDecimal.valueOf(50),
                "FTD", BigDecimal.valueOf(30),
                "FVA", BigDecimal.valueOf(20)
        )));
        mtsMap.put("JPY", new HashMap<>(Map.of(
                "MTD", BigDecimal.valueOf(60),
                "FTD", BigDecimal.valueOf(40),
                "FVA", BigDecimal.valueOf(25)
        )));

        // Perform the aggregation and get the residual maps
        Map<String, BigDecimal> propResidual = new HashMap<>();
        Map<String, BigDecimal> cmggResidual = new HashMap<>();
        aggregateMaps(t69Map, mtsMap, propResidual, cmggResidual);

        // Print the resulting t69Map
        System.out.println("Updated t69Map:");
        t69Map.forEach((currency, innerMap) -> {
            System.out.println("Currency: " + currency);
            innerMap.forEach((key, value) -> System.out.println("  Key: " + key + ", Value: " + value));
        });

        // Print the residual maps
        System.out.println("\nProp Residual:");
        propResidual.forEach((currency, value) -> System.out.println("Currency: " + currency + ", Value: " + value));

        System.out.println("\nCMGG Residual:");
        cmggResidual.forEach((currency, value) -> System.out.println("Currency: " + currency + ", Value: " + value));
    }

    public static void aggregateMaps(Map<String, Map<String, BigDecimal>> t69Map, Map<String, Map<String, BigDecimal>> mtsMap,
                                     Map<String, BigDecimal> propResidual, Map<String, BigDecimal> cmggResidual) {
        mtsMap.forEach((currency, mtsInnerMap) -> {
            // Retrieve or initialize the inner map for the currency in t69Map
            t69Map.putIfAbsent(currency, new HashMap<>());
            Map<String, BigDecimal> t69InnerMap = t69Map.get(currency);

            // Sum MTD and FTD from mtsMap
            BigDecimal mtdValue = mtsInnerMap.getOrDefault("MTD", BigDecimal.ZERO);
            BigDecimal ftdValue = mtsInnerMap.getOrDefault("FTD", BigDecimal.ZERO);
            BigDecimal mtdFtdSum = mtdValue.add(ftdValue);

            // Add the sum to t69Map for key "PROP"
            BigDecimal updatedProp = t69InnerMap.getOrDefault("PROP", BigDecimal.ZERO).add(mtdFtdSum);
            t69InnerMap.put("PROP", updatedProp);

            // Update propResidual map
            propResidual.put(currency, mtdFtdSum);

            // Add FVA to t69Map for key "CMGG"
            BigDecimal fvaValue = mtsInnerMap.getOrDefault("FVA", BigDecimal.ZERO);
            BigDecimal updatedCmgg = t69InnerMap.getOrDefault("CMGG", BigDecimal.ZERO).add(fvaValue);
            t69InnerMap.put("CMGG", updatedCmgg);

            // Update cmggResidual map
            cmggResidual.put(currency, fvaValue);
        });
    }
}
