import java.util.*;
import java.util.stream.Collectors;

class Trade {
    String tradeDate;
    String tradeId;
    String actionDate;
    String action;
    String specialExposureType;
    String tradeStatus; // Field to set the status

    // Constructor, getters, and setters omitted for brevity
}

class Exercise {
    String tradeDate;
    String tradeId;
    String exerciseStatus;

    // Constructor, getters, and setters omitted for brevity
}

public class TradeProcessor {

    public static void main(String[] args) {
        List<Trade> trades = new ArrayList<>(); // Populate with Trade objects
        List<Exercise> exercises = new ArrayList<>(); // Populate with Exercise objects

        String currentDate = "20240920";
        String previousDate = "20240919";

        // Cache previous date tradeIds for efficient lookup
        Set<String> previousDateTradeIds = trades.stream()
                .filter(t -> previousDate.equals(t.tradeDate))
                .map(Trade::getTradeId)
                .collect(Collectors.toSet());

        // Process trades
        trades.forEach(trade -> {
            if (currentDate.equals(trade.tradeDate) &&
                currentDate.equals(trade.actionDate) &&
                "C".equals(trade.action) &&
                "C".equals(trade.specialExposureType)) {
                trade.setTradeStatus("Cancelled");
            } else if (currentDate.equals(trade.tradeDate) &&
                       currentDate.equals(trade.actionDate) &&
                       ("A".equals(trade.action) || "M".equals(trade.action)) &&
                       "C".equals(trade.specialExposureType)) {
                trade.setTradeStatus("Amended");
            } else if (currentDate.equals(trade.tradeDate) &&
                       !previousDateTradeIds.contains(trade.getTradeId()) &&
                       "C".equals(trade.specialExposureType)) {
                trade.setTradeStatus("New");
            }
        });

        // Process trades with exercises for 'New' status
        trades.stream()
              .filter(trade -> currentDate.equals(trade.tradeDate))
              .forEach(trade -> {
                  boolean hasExerciseMatch = exercises.stream()
                          .anyMatch(exercise -> exercise.tradeDate.equals(trade.tradeDate) &&
                                                exercise.tradeId.equals(trade.tradeId) &&
                                                "EXD".equals(exercise.exerciseStatus));
                  if (hasExerciseMatch) {
                      trade.setTradeStatus("New");
                  }
              });

        // Print updated trades for verification
        trades.forEach(trade -> System.out.println(
                "TradeId: " + trade.getTradeId() +
                ", TradeDate: " + trade.getTradeDate() +
                ", TradeStatus: " + trade.getTradeStatus()));
    }
}
