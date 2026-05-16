import javax.swing.*;
import javax.swing.border.*;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;

/**
 * AI Chatbox using Java Swing + Anthropic Claude API
 *
 * Usage:
 *   1. Set your API key: export ANTHROPIC_API_KEY=sk-ant-...
 *   2. Compile:  javac AIChatBox.java
 *   3. Run:      java AIChatBox
 *
 * Or hardcode your API key in the API_KEY constant below.
 */
public class AIChatBox extends JFrame {

    // ── Configuration ────────────────────────────────────────────────────────
    private static final String API_KEY   = System.getenv("ANTHROPIC_API_KEY") != null
                                              ? System.getenv("ANTHROPIC_API_KEY")
                                              : "YOUR_API_KEY_HERE";
    private static final String MODEL     = "claude-sonnet-4-20250514";
    private static final String API_URL   = "https://api.anthropic.com/v1/messages";
    private static final int    MAX_TOKENS = 1024;

    // ── Colors ───────────────────────────────────────────────────────────────
    private static final Color BG_DARK    = new Color(18, 18, 22);
    private static final Color BG_PANEL   = new Color(26, 26, 32);
    private static final Color BG_INPUT   = new Color(36, 36, 44);
    private static final Color ACCENT     = new Color(124, 90, 230);
    private static final Color ACCENT_DIM = new Color(80, 55, 160);
    private static final Color MSG_USER   = new Color(42, 38, 72);
    private static final Color MSG_AI     = new Color(30, 30, 40);
    private static final Color TEXT_WHITE = new Color(235, 232, 255);
    private static final Color TEXT_MUTED = new Color(140, 135, 170);
    private static final Color BORDER_CLR = new Color(55, 50, 80);

    // ── State ────────────────────────────────────────────────────────────────
    private final JPanel    messagesPanel;
    private final JScrollPane scrollPane;
    private final JTextArea inputField;
    private final JButton   sendButton;
    private final StringBuilder conversationHistory = new StringBuilder();

    // ─────────────────────────────────────────────────────────────────────────
    public AIChatBox() {
        super("Claude AI Chat");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(720, 820);
        setMinimumSize(new Dimension(500, 600));
        setLocationRelativeTo(null);

        // ── Root layout ──────────────────────────────────────────────────────
        JPanel root = new JPanel(new BorderLayout());
        root.setBackground(BG_DARK);
        setContentPane(root);

        // ── Header ───────────────────────────────────────────────────────────
        root.add(buildHeader(), BorderLayout.NORTH);

        // ── Messages area ────────────────────────────────────────────────────
        messagesPanel = new JPanel();
        messagesPanel.setLayout(new BoxLayout(messagesPanel, BoxLayout.Y_AXIS));
        messagesPanel.setBackground(BG_DARK);
        messagesPanel.setBorder(new EmptyBorder(16, 16, 16, 16));

        addWelcomeMessage();

        scrollPane = new JScrollPane(messagesPanel);
        scrollPane.setBorder(BorderFactory.createEmptyBorder());
        scrollPane.setBackground(BG_DARK);
        scrollPane.getViewport().setBackground(BG_DARK);
        scrollPane.setVerticalScrollBarPolicy(JScrollPane.VERTICAL_SCROLLBAR_AS_NEEDED);
        scrollPane.setHorizontalScrollBarPolicy(JScrollPane.HORIZONTAL_SCROLLBAR_NEVER);
        styleScrollBar(scrollPane.getVerticalScrollBar());
        root.add(scrollPane, BorderLayout.CENTER);

        // ── Input bar ────────────────────────────────────────────────────────
        JPanel inputBar = new JPanel(new BorderLayout(10, 0));
        inputBar.setBackground(BG_PANEL);
        inputBar.setBorder(new CompoundBorder(
            new MatteBorder(1, 0, 0, 0, BORDER_CLR),
            new EmptyBorder(14, 18, 14, 18)
        ));

        inputField = new JTextArea(3, 1);
        inputField.setBackground(BG_INPUT);
        inputField.setForeground(TEXT_WHITE);
        inputField.setCaretColor(ACCENT);
        inputField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        inputField.setLineWrap(true);
        inputField.setWrapStyleWord(true);
        inputField.setBorder(new CompoundBorder(
            new LineBorder(BORDER_CLR, 1, true),
            new EmptyBorder(10, 14, 10, 14)
        ));

        // Send on Enter, newline on Shift+Enter
        inputField.addKeyListener(new KeyAdapter() {
            @Override public void keyPressed(KeyEvent e) {
                if (e.getKeyCode() == KeyEvent.VK_ENTER && !e.isShiftDown()) {
                    e.consume();
                    sendMessage();
                }
            }
        });

        JScrollPane inputScroll = new JScrollPane(inputField);
        inputScroll.setBorder(BorderFactory.createEmptyBorder());
        inputScroll.setBackground(BG_INPUT);

        sendButton = new JButton("Send ↑");
        sendButton.setBackground(ACCENT);
        sendButton.setForeground(Color.WHITE);
        sendButton.setFont(new Font("Segoe UI", Font.BOLD, 13));
        sendButton.setFocusPainted(false);
        sendButton.setBorderPainted(false);
        sendButton.setOpaque(true);
        sendButton.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        sendButton.setPreferredSize(new Dimension(90, 50));
        sendButton.addActionListener(e -> sendMessage());

        sendButton.addMouseListener(new MouseAdapter() {
            @Override public void mouseEntered(MouseEvent e) { sendButton.setBackground(ACCENT_DIM); }
            @Override public void mouseExited(MouseEvent  e) { sendButton.setBackground(ACCENT); }
        });

        inputBar.add(inputScroll, BorderLayout.CENTER);
        inputBar.add(sendButton,  BorderLayout.EAST);
        root.add(inputBar, BorderLayout.SOUTH);

        setVisible(true);
    }

    // ── Header ───────────────────────────────────────────────────────────────
    private JPanel buildHeader() {
        JPanel header = new JPanel(new BorderLayout());
        header.setBackground(BG_PANEL);
        header.setBorder(new CompoundBorder(
            new MatteBorder(0, 0, 1, 0, BORDER_CLR),
            new EmptyBorder(14, 20, 14, 20)
        ));

        JLabel title = new JLabel("✦  Claude AI");
        title.setForeground(TEXT_WHITE);
        title.setFont(new Font("Segoe UI", Font.BOLD, 16));

        JLabel status = new JLabel("● Connected");
        status.setForeground(new Color(80, 200, 130));
        status.setFont(new Font("Segoe UI", Font.PLAIN, 12));

        header.add(title,  BorderLayout.WEST);
        header.add(status, BorderLayout.EAST);
        return header;
    }

    // ── Welcome message ──────────────────────────────────────────────────────
    private void addWelcomeMessage() {
        addBubble("Hello! I'm Claude. How can I help you today?\n" +
                  "(Press Enter to send, Shift+Enter for a new line)", false);
    }

    // ── Send message flow ─────────────────────────────────────────────────────
    private void sendMessage() {
        String text = inputField.getText().trim();
        if (text.isEmpty()) return;

        inputField.setText("");
        addBubble(text, true);

        conversationHistory.append("{\"role\":\"user\",\"content\":\"")
                           .append(escapeJson(text))
                           .append("\"},");

        sendButton.setEnabled(false);
        inputField.setEnabled(false);

        JLabel typingLabel = addTypingIndicator();

        SwingWorker<String, Void> worker = new SwingWorker<>() {
            @Override protected String doInBackground() {
                return callClaudeAPI();
            }
            @Override protected void done() {
                try {
                    String reply = get();
                    messagesPanel.remove(typingLabel);

                    conversationHistory.append("{\"role\":\"assistant\",\"content\":\"")
                                       .append(escapeJson(reply))
                                       .append("\"},");

                    addBubble(reply, false);
                } catch (Exception ex) {
                    messagesPanel.remove(typingLabel);
                    addBubble("⚠ Error: " + ex.getMessage(), false);
                }
                sendButton.setEnabled(true);
                inputField.setEnabled(true);
                inputField.requestFocus();
                revalidate(); repaint();
                scrollToBottom();
            }
        };
        worker.execute();
    }

    // ── Claude API call ───────────────────────────────────────────────────────
    private String callClaudeAPI() {
        try {
            String history = conversationHistory.toString();
            if (history.endsWith(",")) history = history.substring(0, history.length() - 1);

            String body = "{"
                + "\"model\":\"" + MODEL + "\","
                + "\"max_tokens\":" + MAX_TOKENS + ","
                + "\"messages\":[" + history + "]"
                + "}";

            URL url = new URL(API_URL);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setRequestMethod("POST");
            conn.setRequestProperty("Content-Type",       "application/json");
            conn.setRequestProperty("x-api-key",          API_KEY);
            conn.setRequestProperty("anthropic-version",  "2023-06-01");
            conn.setDoOutput(true);

            try (OutputStream os = conn.getOutputStream()) {
                os.write(body.getBytes(StandardCharsets.UTF_8));
            }

            int status = conn.getResponseCode();
            InputStream is = (status >= 200 && status < 300)
                              ? conn.getInputStream()
                              : conn.getErrorStream();

            StringBuilder sb = new StringBuilder();
            try (BufferedReader br = new BufferedReader(new InputStreamReader(is, StandardCharsets.UTF_8))) {
                String line;
                while ((line = br.readLine()) != null) sb.append(line);
            }

            String json = sb.toString();
            if (status >= 400) return "API Error " + status + ": " + json;

            // Parse: "text":"<value>"
            int idx = json.indexOf("\"text\":\"");
            if (idx < 0) return "Unexpected response: " + json;
            idx += 8;
            StringBuilder result = new StringBuilder();
            for (int i = idx; i < json.length(); i++) {
                char c = json.charAt(i);
                if (c == '"' && json.charAt(i - 1) != '\\') break;
                if (c == '\\' && i + 1 < json.length()) {
                    char next = json.charAt(i + 1);
                    switch (next) {
                        case 'n' -> { result.append('\n'); i++; }
                        case 't' -> { result.append('\t'); i++; }
                        case '"' -> { result.append('"');  i++; }
                        case '\\' -> { result.append('\\'); i++; }
                        default -> result.append(c);
                    }
                } else {
                    result.append(c);
                }
            }
            return result.toString();

        } catch (Exception e) {
            return "Connection error: " + e.getMessage();
        }
    }

    // ── Bubble factory ────────────────────────────────────────────────────────
    private void addBubble(String text, boolean isUser) {
        JPanel row = new JPanel(new FlowLayout(isUser ? FlowLayout.RIGHT : FlowLayout.LEFT, 0, 0));
        row.setBackground(BG_DARK);
        row.setMaximumSize(new Dimension(Integer.MAX_VALUE, Integer.MAX_VALUE));

        JTextArea bubble = new JTextArea(text);
        bubble.setEditable(false);
        bubble.setLineWrap(true);
        bubble.setWrapStyleWord(true);
        bubble.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        bubble.setForeground(TEXT_WHITE);
        bubble.setBackground(isUser ? MSG_USER : MSG_AI);
        bubble.setBorder(new CompoundBorder(
            new LineBorder(isUser ? new Color(90, 70, 160) : BORDER_CLR, 1, true),
            new EmptyBorder(10, 14, 10, 14)
        ));
        bubble.setMaximumSize(new Dimension(500, Integer.MAX_VALUE));

        row.add(bubble);
        messagesPanel.add(row);
        messagesPanel.add(Box.createVerticalStrut(10));
        revalidate();
        repaint();
        scrollToBottom();
    }

    // ── Typing indicator ──────────────────────────────────────────────────────
    private JLabel addTypingIndicator() {
        JLabel label = new JLabel("  Claude is thinking…");
        label.setForeground(TEXT_MUTED);
        label.setFont(new Font("Segoe UI", Font.ITALIC, 13));
        label.setAlignmentX(Component.LEFT_ALIGNMENT);
        label.setBorder(new EmptyBorder(4, 0, 4, 0));
        messagesPanel.add(label);
        revalidate(); repaint();
        scrollToBottom();
        return label;
    }

    // ── Utilities ─────────────────────────────────────────────────────────────
    private void scrollToBottom() {
        SwingUtilities.invokeLater(() -> {
            JScrollBar bar = scrollPane.getVerticalScrollBar();
            bar.setValue(bar.getMaximum());
        });
    }

    private void styleScrollBar(JScrollBar bar) {
        bar.setBackground(BG_DARK);
        bar.setForeground(BORDER_CLR);
        bar.setPreferredSize(new Dimension(6, 0));
    }

    private String escapeJson(String s) {
        return s.replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n")
                .replace("\r", "\\r")
                .replace("\t", "\\t");
    }

    // ── Entry point ───────────────────────────────────────────────────────────
    public static void main(String[] args) {
        // Validate API key before opening window
        if (API_KEY.equals("YOUR_API_KEY_HERE")) {
            JOptionPane.showMessageDialog(null,
                "Please set your Anthropic API key.\n\n" +
                "Option 1 – Environment variable (recommended):\n" +
                "  export ANTHROPIC_API_KEY=sk-ant-...\n\n" +
                "Option 2 – Hardcode in AIChatBox.java:\n" +
                "  private static final String API_KEY = \"sk-ant-...\";",
                "API Key Missing", JOptionPane.WARNING_MESSAGE);
        }

        try {
            UIManager.setLookAndFeel(UIManager.getCrossPlatformLookAndFeelClassName());
        } catch (Exception ignored) {}

        SwingUtilities.invokeLater(AIChatBox::new);
    }
}import javax.swing.*;
import javax.swing.border.*;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;

/**
 * AI Chatbox using Java Swing + Anthropic Claude API
 *
 * Usage:
 *   1. Set your API key: export ANTHROPIC_API_KEY=sk-ant-...
 *   2. Compile:  javac AIChatBox.java
 *   3. Run:      java AIChatBox
 *
 * Or hardcode your API key in the API_KEY constant below.
 */
public class AIChatBox extends JFrame {

    // ── Configuration ────────────────────────────────────────────────────────
    private static final String API_KEY   = System.getenv("ANTHROPIC_API_KEY") != null
                                              ? System.getenv("ANTHROPIC_API_KEY")
                                              : "YOUR_API_KEY_HERE";
    private static final String MODEL     = "claude-sonnet-4-20250514";
    private static final String API_URL   = "https://api.anthropic.com/v1/messages";
    private static final int    MAX_TOKENS = 1024;

    // ── Colors ───────────────────────────────────────────────────────────────
    private static final Color BG_DARK    = new Color(18, 18, 22);
    private static final Color BG_PANEL   = new Color(26, 26, 32);
    private static final Color BG_INPUT   = new Color(36, 36, 44);
    private static final Color ACCENT     = new Color(124, 90, 230);
    private static final Color ACCENT_DIM = new Color(80, 55, 160);
    private static final Color MSG_USER   = new Color(42, 38, 72);
    private static final Color MSG_AI     = new Color(30, 30, 40);
    private static final Color TEXT_WHITE = new Color(235, 232, 255);
    private static final Color TEXT_MUTED = new Color(140, 135, 170);
    private static final Color BORDER_CLR = new Color(55, 50, 80);

    // ── State ────────────────────────────────────────────────────────────────
    private final JPanel    messagesPanel;
    private final JScrollPane scrollPane;
    private final JTextArea inputField;
    private final JButton   sendButton;
    private final StringBuilder conversationHistory = new StringBuilder();

    // ─────────────────────────────────────────────────────────────────────────
    public AIChatBox() {
        super("Claude AI Chat");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(720, 820);
        setMinimumSize(new Dimension(500, 600));
        setLocationRelativeTo(null);

        // ── Root layout ──────────────────────────────────────────────────────
        JPanel root = new JPanel(new BorderLayout());
        root.setBackground(BG_DARK);
        setContentPane(root);

        // ── Header ───────────────────────────────────────────────────────────
        root.add(buildHeader(), BorderLayout.NORTH);

        // ── Messages area ────────────────────────────────────────────────────
        messagesPanel = new JPanel();
        messagesPanel.setLayout(new BoxLayout(messagesPanel, BoxLayout.Y_AXIS));
        messagesPanel.setBackground(BG_DARK);
        messagesPanel.setBorder(new EmptyBorder(16, 16, 16, 16));

        addWelcomeMessage();

        scrollPane = new JScrollPane(messagesPanel);
        scrollPane.setBorder(BorderFactory.createEmptyBorder());
        scrollPane.setBackground(BG_DARK);
        scrollPane.getViewport().setBackground(BG_DARK);
        scrollPane.setVerticalScrollBarPolicy(JScrollPane.VERTICAL_SCROLLBAR_AS_NEEDED);
        scrollPane.setHorizontalScrollBarPolicy(JScrollPane.HORIZONTAL_SCROLLBAR_NEVER);
        styleScrollBar(scrollPane.getVerticalScrollBar());
        root.add(scrollPane, BorderLayout.CENTER);

        // ── Input bar ────────────────────────────────────────────────────────
        JPanel inputBar = new JPanel(new BorderLayout(10, 0));
        inputBar.setBackground(BG_PANEL);
        inputBar.setBorder(new CompoundBorder(
            new MatteBorder(1, 0, 0, 0, BORDER_CLR),
            new EmptyBorder(14, 18, 14, 18)
        ));

        inputField = new JTextArea(3, 1);
        inputField.setBackground(BG_INPUT);
        inputField.setForeground(TEXT_WHITE);
        inputField.setCaretColor(ACCENT);
        inputField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        inputField.setLineWrap(true);
        inputField.setWrapStyleWord(true);
        inputField.setBorder(new CompoundBorder(
            new LineBorder(BORDER_CLR, 1, true),
            new EmptyBorder(10, 14, 10, 14)
        ));

        // Send on Enter, newline on Shift+Enter
        inputField.addKeyListener(new KeyAdapter() {
            @Override public void keyPressed(KeyEvent e) {
                if (e.getKeyCode() == KeyEvent.VK_ENTER && !e.isShiftDown()) {
                    e.consume();
                    sendMessage();
                }
            }
        });

        JScrollPane inputScroll = new JScrollPane(inputField);
        inputScroll.setBorder(BorderFactory.createEmptyBorder());
        inputScroll.setBackground(BG_INPUT);

        sendButton = new JButton("Send ↑");
        sendButton.setBackground(ACCENT);
        sendButton.setForeground(Color.WHITE);
        sendButton.setFont(new Font("Segoe UI", Font.BOLD, 13));
        sendButton.setFocusPainted(false);
        sendButton.setBorderPainted(false);
        sendButton.setOpaque(true);
        sendButton.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        sendButton.setPreferredSize(new Dimension(90, 50));
        sendButton.addActionListener(e -> sendMessage());

        sendButton.addMouseListener(new MouseAdapter() {
            @Override public void mouseEntered(MouseEvent e) { sendButton.setBackground(ACCENT_DIM); }
            @Override public void mouseExited(MouseEvent  e) { sendButton.setBackground(ACCENT); }
        });

        inputBar.add(inputScroll, BorderLayout.CENTER);
        inputBar.add(sendButton,  BorderLayout.EAST);
        root.add(inputBar, BorderLayout.SOUTH);

        setVisible(true);
    }

    // ── Header ───────────────────────────────────────────────────────────────
    private JPanel buildHeader() {
        JPanel header = new JPanel(new BorderLayout());
        header.setBackground(BG_PANEL);
        header.setBorder(new CompoundBorder(
            new MatteBorder(0, 0, 1, 0, BORDER_CLR),
            new EmptyBorder(14, 20, 14, 20)
        ));

        JLabel title = new JLabel("✦  Claude AI");
        title.setForeground(TEXT_WHITE);
        title.setFont(new Font("Segoe UI", Font.BOLD, 16));

        JLabel status = new JLabel("● Connected");
        status.setForeground(new Color(80, 200, 130));
        status.setFont(new Font("Segoe UI", Font.PLAIN, 12));

        header.add(title,  BorderLayout.WEST);
        header.add(status, BorderLayout.EAST);
        return header;
    }

    // ── Welcome message ──────────────────────────────────────────────────────
    private void addWelcomeMessage() {
        addBubble("Hello! I'm Claude. How can I help you today?\n" +
                  "(Press Enter to send, Shift+Enter for a new line)", false);
    }

    // ── Send message flow ─────────────────────────────────────────────────────
    private void sendMessage() {
        String text = inputField.getText().trim();
        if (text.isEmpty()) return;

        inputField.setText("");
        addBubble(text, true);

        conversationHistory.append("{\"role\":\"user\",\"content\":\"")
                           .append(escapeJson(text))
                           .append("\"},");

        sendButton.setEnabled(false);
        inputField.setEnabled(false);

        JLabel typingLabel = addTypingIndicator();

        SwingWorker<String, Void> worker = new SwingWorker<>() {
            @Override protected String doInBackground() {
                return callClaudeAPI();
            }
            @Override protected void done() {
                try {
                    String reply = get();
                    messagesPanel.remove(typingLabel);

                    conversationHistory.append("{\"role\":\"assistant\",\"content\":\"")
                                       .append(escapeJson(reply))
                                       .append("\"},");

                    addBubble(reply, false);
                } catch (Exception ex) {
                    messagesPanel.remove(typingLabel);
                    addBubble("⚠ Error: " + ex.getMessage(), false);
                }
                sendButton.setEnabled(true);
                inputField.setEnabled(true);
                inputField.requestFocus();
                revalidate(); repaint();
                scrollToBottom();
            }
        };
        worker.execute();
    }

    // ── Claude API call ───────────────────────────────────────────────────────
    private String callClaudeAPI() {
        try {
            String history = conversationHistory.toString();
            if (history.endsWith(",")) history = history.substring(0, history.length() - 1);

            String body = "{"
                + "\"model\":\"" + MODEL + "\","
                + "\"max_tokens\":" + MAX_TOKENS + ","
                + "\"messages\":[" + history + "]"
                + "}";

            URL url = new URL(API_URL);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setRequestMethod("POST");
            conn.setRequestProperty("Content-Type",       "application/json");
            conn.setRequestProperty("x-api-key",          API_KEY);
            conn.setRequestProperty("anthropic-version",  "2023-06-01");
            conn.setDoOutput(true);

            try (OutputStream os = conn.getOutputStream()) {
                os.write(body.getBytes(StandardCharsets.UTF_8));
            }

            int status = conn.getResponseCode();
            InputStream is = (status >= 200 && status < 300)
                              ? conn.getInputStream()
                              : conn.getErrorStream();

            StringBuilder sb = new StringBuilder();
            try (BufferedReader br = new BufferedReader(new InputStreamReader(is, StandardCharsets.UTF_8))) {
                String line;
                while ((line = br.readLine()) != null) sb.append(line);
            }

            String json = sb.toString();
            if (status >= 400) return "API Error " + status + ": " + json;

            // Parse: "text":"<value>"
            int idx = json.indexOf("\"text\":\"");
            if (idx < 0) return "Unexpected response: " + json;
            idx += 8;
            StringBuilder result = new StringBuilder();
            for (int i = idx; i < json.length(); i++) {
                char c = json.charAt(i);
                if (c == '"' && json.charAt(i - 1) != '\\') break;
                if (c == '\\' && i + 1 < json.length()) {
                    char next = json.charAt(i + 1);
                    switch (next) {
                        case 'n' -> { result.append('\n'); i++; }
                        case 't' -> { result.append('\t'); i++; }
                        case '"' -> { result.append('"');  i++; }
                        case '\\' -> { result.append('\\'); i++; }
                        default -> result.append(c);
                    }
                } else {
                    result.append(c);
                }
            }
            return result.toString();

        } catch (Exception e) {
            return "Connection error: " + e.getMessage();
        }
    }

    // ── Bubble factory ────────────────────────────────────────────────────────
    private void addBubble(String text, boolean isUser) {
        JPanel row = new JPanel(new FlowLayout(isUser ? FlowLayout.RIGHT : FlowLayout.LEFT, 0, 0));
        row.setBackground(BG_DARK);
        row.setMaximumSize(new Dimension(Integer.MAX_VALUE, Integer.MAX_VALUE));

        JTextArea bubble = new JTextArea(text);
        bubble.setEditable(false);
        bubble.setLineWrap(true);
        bubble.setWrapStyleWord(true);
        bubble.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        bubble.setForeground(TEXT_WHITE);
        bubble.setBackground(isUser ? MSG_USER : MSG_AI);
        bubble.setBorder(new CompoundBorder(
            new LineBorder(isUser ? new Color(90, 70, 160) : BORDER_CLR, 1, true),
            new EmptyBorder(10, 14, 10, 14)
        ));
        bubble.setMaximumSize(new Dimension(500, Integer.MAX_VALUE));

        row.add(bubble);
        messagesPanel.add(row);
        messagesPanel.add(Box.createVerticalStrut(10));
        revalidate();
        repaint();
        scrollToBottom();
    }

    // ── Typing indicator ──────────────────────────────────────────────────────
    private JLabel addTypingIndicator() {
        JLabel label = new JLabel("  Claude is thinking…");
        label.setForeground(TEXT_MUTED);
        label.setFont(new Font("Segoe UI", Font.ITALIC, 13));
        label.setAlignmentX(Component.LEFT_ALIGNMENT);
        label.setBorder(new EmptyBorder(4, 0, 4, 0));
        messagesPanel.add(label);
        revalidate(); repaint();
        scrollToBottom();
        return label;
    }

    // ── Utilities ─────────────────────────────────────────────────────────────
    private void scrollToBottom() {
        SwingUtilities.invokeLater(() -> {
            JScrollBar bar = scrollPane.getVerticalScrollBar();
            bar.setValue(bar.getMaximum());
        });
    }

    private void styleScrollBar(JScrollBar bar) {
        bar.setBackground(BG_DARK);
        bar.setForeground(BORDER_CLR);
        bar.setPreferredSize(new Dimension(6, 0));
    }

    private String escapeJson(String s) {
        return s.replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n")
                .replace("\r", "\\r")
                .replace("\t", "\\t");
    }

    // ── Entry point ───────────────────────────────────────────────────────────
    public static void main(String[] args) {
        // Validate API key before opening window
        if (API_KEY.equals("YOUR_API_KEY_HERE")) {
            JOptionPane.showMessageDialog(null,
                "Please set your Anthropic API key.\n\n" +
                "Option 1 – Environment variable (recommended):\n" +
                "  export ANTHROPIC_API_KEY=sk-ant-...\n\n" +
                "Option 2 – Hardcode in AIChatBox.java:\n" +
                "  private static final String API_KEY = \"sk-ant-...\";",
                "API Key Missing", JOptionPane.WARNING_MESSAGE);
        }

        try {
            UIManager.setLookAndFeel(UIManager.getCrossPlatformLookAndFeelClassName());
        } catch (Exception ignored) {}

        SwingUtilities.invokeLater(AIChatBox::new);
    }
}
