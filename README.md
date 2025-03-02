# MULTITHREADED-FILE-COMPRESSION-TOOL
 // MULTITHREADED
 //FILE COMPRESSION
 //TOOL
 #include <iostream>
#include <fstream>
#include <string>
#include <queue>
#include <unordered_map>
#include <vector>

// Node structure for Huffman Tree
struct HuffmanNode {
    char data;
    unsigned frequency;
    HuffmanNode *left, *right;
    
    HuffmanNode(char data, unsigned frequency) {
        this->data = data;
        this->frequency = frequency;
        left = right = nullptr;
    }
};

// Comparison class for priority queue
struct Compare {
    bool operator()(HuffmanNode* l, HuffmanNode* r) {
        return (l->frequency > r->frequency);
    }
};

// Function to build the Huffman Tree
HuffmanNode* buildHuffmanTree(const std::string& text) {
    // Count frequency of characters
    std::unordered_map<char, unsigned> freq;
    for (char c : text) {
        freq[c]++;
    }
    
    // Create a min heap using priority queue
    std::priority_queue<HuffmanNode*, std::vector<HuffmanNode*>, Compare> minHeap;
    
    // Create a leaf node for each character and add to minHeap
    for (auto pair : freq) {
        minHeap.push(new HuffmanNode(pair.first, pair.second));
    }
    
    // Build Huffman Tree
    while (minHeap.size() > 1) {
        // Extract the two nodes with lowest frequencies
        HuffmanNode* left = minHeap.top(); minHeap.pop();
        HuffmanNode* right = minHeap.top(); minHeap.pop();
        
        // Create a new internal node with frequency equal to sum
        // of the two nodes. '$' is a special value for internal nodes
        HuffmanNode* top = new HuffmanNode('$', left->frequency + right->frequency);
        top->left = left;
        top->right = right;
        
        minHeap.push(top);
    }
    
    // Return root of Huffman Tree
    return minHeap.top();
}

// Recursive function to generate codes and store in codes map
void generateCodes(HuffmanNode* root, const std::string& code, 
                  std::unordered_map<char, std::string>& codes) {
    if (!root) return;
    
    // If this is a leaf node, store the code
    if (!root->left && !root->right) {
        codes[root->data] = code;
    }
    
    // Traverse left with code + "0"
    generateCodes(root->left, code + "0", codes);
    
    // Traverse right with code + "1"
    generateCodes(root->right, code + "1", codes);
}

// Function to compress a file
void compressFile(const std::string& inputFile, const std::string& outputFile) {
    // Read input file
    std::ifstream inFile(inputFile, std::ios::binary);
    if (!inFile) {
        std::cerr << "Cannot open input file: " << inputFile << std::endl;
        return;
    }
    
    std::string text((std::istreambuf_iterator<char>(inFile)),
                      std::istreambuf_iterator<char>());
    inFile.close();
    
    if (text.empty()) {
        std::cerr << "Input file is empty!" << std::endl;
        return;
    }
    
    // Build Huffman Tree
    HuffmanNode* root = buildHuffmanTree(text);
    
    // Generate codes for each character
    std::unordered_map<char, std::string> codes;
    generateCodes(root, "", codes);
    
    // Output file (compressed)
    std::ofstream outFile(outputFile, std::ios::binary);
    if (!outFile) {
        std::cerr << "Cannot open output file: " << outputFile << std::endl;
        return;
    }
    
    // Write the character codes map for decompression
    outFile << codes.size() << std::endl;
    for (auto pair : codes) {
        // Store character, its ASCII code and Huffman code
        outFile << pair.first << " " << (int)pair.first << " " << pair.second << std::endl;
    }
    
    // Encode the text
    std::string encodedText = "";
    for (char c : text) {
        encodedText += codes[c];
    }
    
    // Add padding bits to make the total bits a multiple of 8
    int padding = 8 - (encodedText.length() % 8);
    if (padding == 8) padding = 0;
    
    outFile << padding << std::endl;
    
    // Convert binary string to bytes and write to file
    std::string bytesStr = "";
    for (int i = 0; i < encodedText.length(); i += 8) {
        std::string byte = "";
        for (int j = 0; j < 8 && i + j < encodedText.length(); j++) {
            byte += encodedText[i + j];
        }
        
        // Pad the last byte if needed
        while (byte.length() < 8) {
            byte += "0";
        }
        
        // Convert byte string to a char
        char c = 0;
        for (int j = 0; j < 8; j++) {
            if (byte[j] == '1') {
                c |= (1 << (7 - j));
            }
        }
        
        bytesStr += c;
    }
    
    outFile << bytesStr;
    outFile.close();
    
    std::cout << "File compressed successfully to " << outputFile << std::endl;
    
    // Clean up Huffman Tree
    // (implement a proper cleanup function to avoid memory leaks)
}

// Function to decompress a file
void decompressFile(const std::string& inputFile, const std::string& outputFile) {
    // Read input file (compressed)
    std::ifstream inFile(inputFile, std::ios::binary);
    if (!inFile) {
        std::cerr << "Cannot open input file: " << inputFile << std::endl;
        return;
    }
    
    // Read the codes map
    int codesSize;
    inFile >> codesSize;
    inFile.ignore(); // Skip the newline
    
    std::unordered_map<std::string, char> reverseCodes;
    for (int i = 0; i < codesSize; i++) {
        char c;
        int asciiCode;
        std::string code;
        
        inFile >> c >> asciiCode >> code;
        inFile.ignore(); // Skip the newline
        
        reverseCodes[code] = c;
    }
    
    // Read padding
    int padding;
    inFile >> padding;
    inFile.ignore(); // Skip the newline
    
    // Read the encoded data
    std::string encodedData((std::istreambuf_iterator<char>(inFile)),
                             std::istreambuf_iterator<char>());
    inFile.close();
    
    // Convert bytes to binary string
    std::string binaryString = "";
    for (char c : encodedData) {
        for (int i = 7; i >= 0; i--) {
            binaryString += ((c & (1 << i)) ? '1' : '0');
        }
    }
    
    // Remove padding bits
    if (padding > 0) {
        binaryString = binaryString.substr(0, binaryString.length() - padding);
    }
    
    // Decode the binary string
    std::ofstream outFile(outputFile, std::ios::binary);
    if (!outFile) {
        std::cerr << "Cannot open output file: " << outputFile << std::endl;
        return;
    }
    
    std::string currentCode = "";
    for (char bit : binaryString) {
        currentCode += bit;
        if (reverseCodes.find(currentCode) != reverseCodes.end()) {
            outFile << reverseCodes[currentCode];
            currentCode = "";
        }
    }
    
    outFile.close();
    
    std::cout << "File decompressed successfully to " << outputFile << std::endl;
}

// Function to clean up Huffman Tree (prevent memory leaks)
void cleanUpTree(HuffmanNode* root) {
    if (root == nullptr) return;
    
    cleanUpTree(root->left);
    cleanUpTree(root->right);
    
    delete root;
}

// Main function with command line interface
int main(int argc, char* argv[]) {
    if (argc < 4) {
        std::cout << "Usage: " << std::endl;
        std::cout << "  Compress:   " << argv[0] << " -c <input_file> <output_file>" << std::endl;
        std::cout << "  Decompress: " << argv[0] << " -d <input_file> <output_file>" << std::endl;
        return 1;
    }
    
    std::string mode = argv[1];
    std::string inputFile = argv[2];
    std::string outputFile = argv[3];
    
    if (mode == "-c") {
        std::cout << "Compressing " << inputFile << " to " << outputFile << std::endl;
        compressFile(inputFile, outputFile);
    } else if (mode == "-d") {
        std::cout << "Decompressing " << inputFile << " to " << outputFile << std::endl;
        decompressFile(inputFile, outputFile);
    } else {
        std::cout << "Invalid mode. Use -c for compression or -d for decompression." << std::endl;
        return 1;
    }
    
    return 0;
}
