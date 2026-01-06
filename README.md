# threedassets

A new Flutter project.

## Getting Started

This project is a starting point for a Flutter application.

A few resources to get you started if this is your first Flutter project:

- [Lab: Write your first Flutter app](https://docs.flutter.dev/get-started/codelab)
- [Cookbook: Useful Flutter samples](https://docs.flutter.dev/cookbook)

For help getting started with Flutter development, view the
[online documentation](https://docs.flutter.dev/), which offers tutorials,
samples, guidance on mobile development, and a full API reference.

import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';
import 'package:flutter_html/flutter_html.dart';
import 'package:flutter_screenutil/flutter_screenutil.dart';
import 'package:flutter_svg/svg.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:foodking/core/extensions/string_extensions.dart';
import 'package:foodking/translation/locale_keys.g.dart';
import 'dart:async';
import '../app/modules/home/logic/home_cubit.dart';
import 'package:shimmer/shimmer.dart';
import '../app/modules/home/logic/home_state.dart';
import '../app/modules/cart/cubit/cart_cubit.dart';
import '../app/modules/cart/cubit/cart_state.dart';
import 'item_caution.dart';
import '../app/modules/item/views/item_view.dart';
import '../util/constant.dart';
import '../util/style.dart';
import 'item_quick_change_dialog.dart';

class ItemCardGrid extends StatefulWidget {
  final dynamic item;
  final int index;

  const ItemCardGrid({super.key, required this.item, required this.index});

  @override
  State<ItemCardGrid> createState() => _ItemCardGridState();
}

class _ItemCardGridState extends State<ItemCardGrid> {
  Timer? _autoCloseTimer;
  bool _showQuantityControls = false;

  @override
  void dispose() {
    _autoCloseTimer?.cancel();
    super.dispose();
  }

  void _startAutoCloseTimer() {
    _autoCloseTimer?.cancel();
    _autoCloseTimer = Timer(const Duration(seconds: 2), () {
      if (mounted) {
        setState(() {
          _showQuantityControls = false;
        });
      }
    });
  }

  // Optimistic UI state
  int? _localQuantity;
  Timer? _debounceTimer;
  bool _isApiLoading = false;
  int? _startQuantity; // Track starting quantity for delta calculation

  void _incrementQuantity(int currentQty) {
    // If local quantity is not set, initialize it from current display logic
    if (_localQuantity == null) {
      _startQuantity = currentQty;
      _localQuantity = currentQty;
    }
    
    setState(() {
      _localQuantity = (_localQuantity ?? 0) + 1;
      _showQuantityControls = true;
    });
    
    _scheduleApiCall();
    _startAutoCloseTimer();
  }

  void _decrementQuantity(int currentQty, String? rowId) {
    if ((_localQuantity ?? currentQty) > 0) {
      if (_localQuantity == null) {
        _startQuantity = currentQty;
        _localQuantity = currentQty;
      }

      setState(() {
        int newQty = (_localQuantity ?? 0) - 1;
        _localQuantity = newQty < 0 ? 0 : newQty;
        
        // If quantity becomes 0, wait for debounce to remove
        if (_localQuantity == 0) {
           _showQuantityControls = false;
           _autoCloseTimer?.cancel();
        } else {
           _startAutoCloseTimer();
        }
      });
      
      _scheduleApiCall(rowId: rowId);
    }
  }

  void _scheduleApiCall({String? rowId}) {
    _debounceTimer?.cancel();
    _debounceTimer = Timer(const Duration(seconds: 2), () async {
      await _syncQuantity(rowId);
    });
  }

  Future<void> _syncQuantity(String? rowId) async {
    if (_localQuantity == null) return;
    
    setState(() {
      _isApiLoading = true;
    });

    try {
      final itemData = widget.item[widget.index];
      final targetQty = _localQuantity!;
      final startQty = _startQuantity ?? 0; // Quantity when we STARTED editing

      if (targetQty == 0) {
         // Remove item
         if (rowId != null) {
            await context.read<CartCubit>().removeCartItem(rowId);
         } else {
             // Fallback using update to 0
            await context.read<CartCubit>().updateCartItemQuantity(itemData.id!, 0);
         }
      } else {
         // If we started from 0 (item not in cart), we must use addToCart
         if (startQty == 0) {
           // We are adding a NEW item with quantity = targetQty
           await context.read<CartCubit>().addToCartApi(
             id: itemData.id!,
             qty: targetQty,
           );
         } else {
            // Item was already in cart, use Update (PUT)
             await context.read<CartCubit>().updateCartItemQuantity(
                itemData.id!,
                targetQty,
             );
         }
      }
    } catch (e) {
      print("Error syncing quantity: $e");
    } finally {
      if (mounted) {
        setState(() {
          _isApiLoading = false;
          _localQuantity = null; // Revert to Cart state source of truth
          _startQuantity = null;
        });
      }
    }
  }

  void _openItemView() {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      backgroundColor: Colors.transparent,
      builder: (context) => ItemView(
        itemDetails: widget.item[widget.index],
        slug: widget.item[widget.index].slug,
        indexNumber: widget.index,
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final itemData = widget.item[widget.index];
    final hasVariations = itemData.isVariant == true;

    return BlocBuilder<CartCubit, CartState>(
      builder: (context, cartState) {
        // Get quantity from cart if item exists in cart
        int cartQuantity = 0;
        String? matchedRowId;
        
        if (cartState is CartLoaded && cartState.cartItems.isNotEmpty) {
          for (var cartItem in cartState.cartItems) {
            bool isMatch = cartItem.id == itemData.id ||
                          (cartItem.name != null && itemData.name != null && 
                           cartItem.name!.trim() == itemData.name!.trim());
            if (isMatch) {
              cartQuantity += (cartItem.qty ?? 0);
              // Store the first matching rowId for potential removal
              if (matchedRowId == null) matchedRowId = cartItem.rowId;
            }
          }
        }

        // Use cart quantity as the source of truth
        // Use local quantity if we are editing, otherwise use cart quantity
        final displayQuantity = _localQuantity ?? cartQuantity;

        return _buildCard(context, itemData, hasVariations, displayQuantity, matchedRowId);
      },
    );
  }

  Widget _buildCard(BuildContext context, dynamic itemData, bool hasVariations, int displayQuantity, String? matchedRowId) {

    return InkWell(
      onTap: hasVariations ? _openItemView : null,
      child: Container(
        height: 220.h, // Adjusted height for better layout
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(16.r),
          color: Colors.white,
        ),
        child: Column(
          children: [
            // Image Stack
            Stack(
              children: [
                Container(
                  height: 150.h,
                  width: double.infinity,
                  decoration: BoxDecoration(
                    borderRadius: BorderRadius.circular(16.r),
                    border: Border.all(color: Colors.grey.shade200),
                  ),
                  child: ClipRRect(
                    borderRadius: BorderRadius.circular(16.r),
                    child: CachedNetworkImage(
                      imageUrl: itemData.imageUrl ?? itemData.cover ?? "",
                      imageBuilder: (context, imageProvider) => Container(
                        decoration: BoxDecoration(
                          image: DecorationImage(
                            image: imageProvider,
                            fit: BoxFit.cover,
                          ),
                        ),
                      ),
                      placeholder: (context, url) => Shimmer.fromColors(
                        child: Container(height: 150.h, width: double.infinity, color: Colors.grey),
                        baseColor: Colors.grey[300]!,
                        highlightColor: Colors.grey[400]!,
                      ),
                      errorWidget: (context, url, error) => const Icon(Icons.error),
                    ),
                  ),
                ),
                
                // Add Button Overlay
                Positioned(
                   bottom: 8.h,
                   left: 8.w,
                   child: _buildAddButton(hasVariations, displayQuantity, matchedRowId),
                ),
              ],
            ),
            
            Expanded(
              child: Padding(
                padding: EdgeInsets.symmetric(horizontal: 8.w, vertical: 8.h),
                child: Column(
                   crossAxisAlignment: CrossAxisAlignment.start,
                   mainAxisAlignment: MainAxisAlignment.start,
                   children: [
                      // Title with Caution Icon
                      Row(
                        mainAxisAlignment: MainAxisAlignment.spaceBetween,
                        children: [
                          Expanded(
                            child: Text(
                              itemData.name!,
                              style: fontBold.copyWith(fontSize: 14.sp),
                              maxLines: 1,
                              overflow: TextOverflow.ellipsis,
                            ),
                          ),
                          if (itemData.description != null && itemData.description!.isNotEmpty)
                          InkWell(
                              onTap: () {
                                showBottomSheet(
                                  context: context,
                                  backgroundColor: Colors.transparent,
                                  builder: (context) => SingleChildScrollView(
                                    child: ItemCaution(
                                      itemName: itemData.name,
                                      itemCaution: itemData.description,
                                    ),
                                  ),
                                );
                              },
                              child: Padding(
                                padding: EdgeInsets.only(left: 4.w),
                                child: SvgPicture.asset(
                                  Images.iconDetails,
                                  width: 16.w,
                                  height: 16.h,
                                  fit: BoxFit.cover,
                                ),
                              ),
                           )
                        ],
                      ),
                       
                       SizedBox(height: 4.h),

                       // Price
                      Row(
                        mainAxisAlignment: MainAxisAlignment.start,
                        children: [
                        itemData.offer.isNotEmpty
                            ? Row(
                                children: [
                                  Text(
                                    itemData.formattedPrice,
                                    style: TextStyle(
                                      fontFamily: 'Rubik',
                                      fontWeight: FontWeight.w500,
                                      fontSize: 10.sp,
                                      decoration: TextDecoration.lineThrough,
                                      color: AppColor.gray,
                                    ),
                                  ),
                                  SizedBox(width: 4.w),
                                  Text(
                                    itemData.offer[0].formattedPrice,
                                    style: fontMediumProWithCurrency,
                                  ),
                                ],
                              )
                            : Text(
                                itemData.formattedPrice,
                                style: fontMediumProWithCurrency.copyWith(fontSize: 12.sp, fontWeight: FontWeight.w500),
                              ),
                        ],
                      ),
                   ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildAddButton(bool hasVariations, int displayQuantity, String? matchedRowId) {
    if (displayQuantity == 0) {
      // Show arrow for variants, + for direct add
      return InkWell(
        onTap: () {
          if (hasVariations) {
            _openItemView();
          } else {
            _incrementQuantity(displayQuantity);
          }
        },
        child: Container(
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            color: Colors.white,
            border: Border.all(
              color: AppColor.gray.withOpacity(0.3),
            ),
            boxShadow: [
              BoxShadow(
                color: Colors.black.withOpacity(0.12), // shadow color
                blurRadius: 6,                         // softness
                spreadRadius: 1,                       // size
                offset: const Offset(0, 2),             // position (x, y)
              ),
            ],
          ),
          alignment: Alignment.center,
          child: Padding(
            padding: const EdgeInsets.all(5.0),
            child: Icon(
              hasVariations ? Icons.chevron_right : Icons.add,
              color: AppColor.primaryColor,
              size: 20,
            ),
          ),
        ),

      );
    }

    if (_showQuantityControls && displayQuantity > 0) {
      return Container(
        padding: EdgeInsets.symmetric(horizontal: 12.w, vertical: 8.h),
        decoration: BoxDecoration(
          color: Colors.white,
          borderRadius: BorderRadius.circular(28.r),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.1),
              blurRadius: 4,
            ),
          ],
        ),
        child: _isApiLoading 
        ? SizedBox(
            width: 24.w, 
            height: 24.w, 
            child: CircularProgressIndicator(strokeWidth: 2, color: AppColor.primaryColor)
          )
        : Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            InkWell(
              onTap: () => _decrementQuantity(displayQuantity, matchedRowId),
              child: Icon(Icons.remove_circle, color: AppColor.primaryColor, size: 28.w),
            ),
            SizedBox(width: 10.w),
            Text("$displayQuantity", style: fontBold.copyWith(fontSize: 16.sp)),
            SizedBox(width: 10.w),
            InkWell(
              onTap: () => _incrementQuantity(displayQuantity),
              child: Icon(Icons.add_circle, color: AppColor.primaryColor, size: 28.w),
            ),
          ],
        ),
      );
    }

    // Show quantity badge (orange circle)
    return InkWell(
      onTap: () {
        if (hasVariations) {
          showModalBottomSheet(
            context: context,
            backgroundColor: Colors.transparent,
            isScrollControlled: true,
            builder: (context) => ItemQuickChangeDialog(
               itemData: widget.item[widget.index],
               onAddNew: _openItemView,
            ),
          );
        } else {
          // No variants - show quantity controls
          setState(() {
            _showQuantityControls = true;
          });
          _startAutoCloseTimer();
        }
      },
      child: Container(
        width: 36.w,
        height: 36.w,
        decoration: BoxDecoration(
          color: AppColor.primaryColor,
          shape: BoxShape.circle,
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.1),
              blurRadius: 4,
            ),
          ],
        ),
        child: Center(
          child: Text(
            "$displayQuantity",
            style: fontBold.copyWith(
              fontSize: 14.sp,
              color: Colors.white,
            ),
          ),
        ),
      ),
    );
  }
}

// Helper function to maintain backward compatibility
Widget itemCardGrid(dynamic item, int index, BuildContext context) {
  return ItemCardGrid(item: item, index: index);
}
